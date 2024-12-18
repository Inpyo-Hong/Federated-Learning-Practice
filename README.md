# Federated Learning Practice


## 기존 Machine Learning 시스템의 한계
각 User들이 Data를 중앙집중화된 서버에 Merge하고, Server는 Model을 학습시킨 후 User들에게 추론결과를 전달하는 방식이다.
하지만 User들이 Sensitive data(ex.medical data, financial data)를 Server에 전달할 경우 privacy 및 법적 문제가 생길 수 있다.
또한, 다수의 User들이 Server를 동시에 점유하는 경우 실시간 시스템의 동작에 한계가 생긴다.

## Federated Learning(연합학습)이란

각 환경에 맞는 Model을 다른 User들과의 연합시스템을 구성하여 학습하는 방식으로, 기존 Machine Learning의 한계를 개선하기 위해 나온 방법이다.
연합학습의 가장 큰 장점은 각 User가 자신의 Device에 있는 데이터를 유출시키지 않으면서, 서버에 존재하는 하나의 Model을 함께 학습시킬 수 있다는 것이다.
동작방식은 여러 User가 각자의 데이터로 학습을 시킨 Model을 Server에 올려보냄으로써 서버에 모인 여러 Model값을 기반으로 Cloud에서 Globalization과정을 거친다.



## 1. Pytorch 설치 및 가상환경 설치

virtual env name = test_fed

python version = 3.8

## 2. 가상환경 requirements 설치

torch

torchvision

flower

pip

tqdm

![image](https://user-images.githubusercontent.com/64252911/191291690-960d5f38-19c7-4a05-8aed-bd2ad1e82ae8.png)


## 3. Server 실행

시간관계 상 num_round는 10으로 설정하였음.

    def weighted_average(metrics: List[Tuple[int, Metrics]]) -> Metrics:
        # Multiply accuracy of each client by number of examples used
        accuracies = [num_examples * m["accuracy"] for num_examples, m in metrics]
        examples = [num_examples for num_examples, _ in metrics]

        # Aggregate and return custom metric (weighted average)
        print("accuracy : {}".format(sum(accuracies) / sum(examples)))
        return {"accuracy": sum(accuracies) / sum(examples)}

    # Define strategy
    # client_manager = SimpleClientManager()
    strategy = flwr.server.strategy.FedAvg(min_fit_clients=3, min_available_clients=3, evaluate_metrics_aggregation_fn=weighted_average)
    # server = Server(client_manager=client_manager, strategy=strategy)

    # Start Flower server
    fl.server.start_server(
        server_address="0.0.0.0:8080",
        config=fl.server.ServerConfig(num_rounds=10),
        strategy=strategy,
    )
    
    
    
![image](https://user-images.githubusercontent.com/64252911/191292852-3a8c8333-fae8-4f2f-b1a4-b6ec9899c95f.png)

## 4. Client 실행

    import warnings

    from tqdm import tqdm
    from collections import OrderedDict

    import flwr as fl
    import torch
    import torch.nn as nn
    import torchvision.transforms as transforms
    from torch.utils.data import Dataset, DataLoader
    from torch.multiprocessing import Process
    import numpy as np
    import json
    import random

    warnings.filterwarnings("ignore", category=UserWarning)

    class FemnistDataset(Dataset):
        def __init__(self, dataset, transform):
            self.x = dataset['x']
            self.y = dataset['y']
            self.transform = transform

      def __getitem__(self, index):
          input_data = np.array(self.x[index]).reshape(28,28,1)
          if self.transform:
              input_data = self.transform(input_data)
          target_data = self.y[index]
          return input_data, target_data

      def __len__(self):
          return len(self.y)

    class femnist_network(nn.Module):
        def __init__(self) -> None:
            super(femnist_network, self).__init__()
            self.conv1 = nn.Conv2d(in_channels=1, out_channels=32, kernel_size=5, padding=2)
            self.maxpool1 = nn.MaxPool2d(kernel_size=(2, 2))
            self.conv2 = nn.Conv2d(in_channels=32, out_channels=64, kernel_size=5, padding=2)
            self.maxpool2 = nn.MaxPool2d(kernel_size=(2, 2))
            self.linear1 = nn.Linear(7*7*64, 2048)
            self.linear2 = nn.Linear(2048, 62)

      def forward(self, x:torch.Tensor) -> torch.Tensor:
          x = torch.relu(self.conv1(x))
          x = self.maxpool1(x)
          x = torch.relu(self.conv2(x))
          x = self.maxpool2(x)
          x = torch.flatten(x, start_dim=1)
          x = torch.relu((self.linear1(x)))
          x = self.linear2(x)
          return x

    def main(DEVICE):
        """Create model, load data, define Flower client, start Flower client."""
        DEVICE = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")
        net = femnist_network().to(DEVICE)

      def load_data():
          """Load CIFAR-10 (training and test set)."""
          transform = transforms.Compose(
              [transforms.ToTensor()]
          )
          number = random.randint(0, 35)
          if number == 35:
              subject_number = random.randint(0, 96)
          else:
              subject_number = random.randint(0, 99)
          print('number : {}, subject number : {}'.format(number, subject_number))
          with open("./data/data/train/all_data_"+str(number)+"_niid_0_keep_0_train_9.json","r") as f:
              train_json = json.load(f)
          with open("./data/data/test/all_data_"+str(number)+"_niid_0_keep_0_test_9.json","r") as f:
              test_json = json.load(f)
          train_user = train_json['users'][subject_number]
          train_data = train_json['user_data'][train_user]
          test_user = test_json['users'][subject_number]
          test_data = test_json['user_data'][test_user]
          trainset = FemnistDataset(train_data, transform)
          testset = FemnistDataset(test_data, transform)
          trainloader = DataLoader(trainset, batch_size=64, shuffle=True)
          testloader = DataLoader(testset, batch_size=64)
          return trainloader, testloader

        class CifarClient(fl.client.NumPyClient):
            def get_parameters(self, config):
                return [val.cpu().numpy() for _, val in net.state_dict().items()]

          def set_parameters(self, parameters):
              params_dict = zip(net.state_dict().keys(), parameters)
              state_dict = OrderedDict({k: torch.Tensor(v) for k, v in params_dict})
              net.load_state_dict(state_dict, strict=True)

          def fit(self, parameters, config):
              self.set_parameters(parameters)
              trainloader, _ = load_data()
              train(net, trainloader, epochs=5)
              return self.get_parameters(config={}), len(trainloader.dataset), {}

          def evaluate(self, parameters, config):
              self.set_parameters(parameters)
              _, testloader = load_data()
              loss, accuracy = test(net, testloader)
              return float(loss), len(testloader.dataset), {"accuracy": accuracy}

      def train(net, trainloader, epochs):
          """Train the network on the training set."""
          criterion = torch.nn.CrossEntropyLoss()
          optimizer = torch.optim.SGD(net.parameters(), lr=0.0003)
          net.train()
          for _ in range(epochs):
              for images, labels in trainloader:
                  images, labels = images.to(DEVICE).float(), labels.to(DEVICE)
                  optimizer.zero_grad()
                  loss = criterion(net(images), labels)
                  loss.backward()
                  optimizer.step()

      def test(net, testloader):
          """Validate the model on the test set."""
          criterion = torch.nn.CrossEntropyLoss()
          correct, total, loss = 0, 0, 0.0
          with torch.no_grad():
              for images, labels in tqdm(testloader):
                  outputs = net(images.to(DEVICE))
                  labels = labels.to(DEVICE)
                  loss += criterion(outputs, labels).item()
                  total += labels.size(0)
                  correct += (torch.max(outputs.data, 1)[1] == labels).sum().item()
          return loss / len(testloader.dataset), correct / total

      def test(net, testloader):
          """Validate the network on the entire test set."""
          criterion = torch.nn.CrossEntropyLoss()
          correct, total, loss = 0, 0, 0.0
          net.eval()
          with torch.no_grad():
              for data in testloader:
                  images, labels = data[0].to(DEVICE).float(), data[1].to(DEVICE)
                  outputs = net(images)
                  loss += criterion(outputs, labels).item()
                  _, predicted = torch.max(outputs.data, 1)
                  total += labels.size(0)
                  correct += (predicted == labels).sum().item()
          accuracy = correct / total
          return loss, accuracy

      # Start client
      fl.client.start_numpy_client(server_address="127.0.0.1:8080", client=CifarClient())

    if __name__ == "__main__":
      torch.multiprocessing.set_start_method('spawn')
      list = [1,2,3]

      ps = []
      for i in list:
          p =Process(target=main, args=(i, ))
          ps.append(p)
          p.start()
      for p in ps:
          p.join()
          
![image](https://user-images.githubusercontent.com/64252911/191296764-680d4858-7227-4900-a18c-6a3d6ea7ed26.png)


## 5. 실험결과

핵심 실험결과는 다음과 같음.

    accuracy : 0.0
    accuracy : 0.06
    accuracy : 0.0375
    accuracy : 0.08433734939759036
    accuracy : 0.08450704225352113
    accuracy : 0.06060606060606061
    accuracy : 0.024096385542168676
    accuracy : 0.046153846153846156
    accuracy : 0.019230769230769232
    accuracy : 0.022727272727272728
    INFO flower 2022-09-21 00:00:54,337 | app.py:180 | app_fit: losses_distributed [(1, 4.131128208977835), (2, 4.1253755664825436), (3, 4.117458188533783), (4, 4.113470743937665), (5, 4.106676350177174), (6, 4.105108102162679), (7, 4.110513589468347), (8, 4.100500906430758), (9, 4.110632786383996), (10, 4.108614439314062)]
    INFO flower 2022-09-21 00:00:54,337 | app.py:181 | app_fit: metrics_distributed {'accuracy': [(1, 0.0), (2, 0.06), (3, 0.0375), (4, 0.08433734939759036), (5, 0.08450704225352113), (6, 0.06060606060606061), (7, 0.024096385542168676), (8, 0.046153846153846156), (9, 0.019230769230769232), (10, 0.022727272727272728)]}

 
 ## etc
 
git 업로드 문제 상 실험에 사용한 데이터셋은 제외 후 push 하였음.

## review

AI의 기본적인 platform구축을 실습함으로써 많은 도움이 되었으며,
연합학습(Federated Learning)을 FEMNIST 데이터셋을 통해 실습함으로 기초 지식 및 running 구조를 보다 쉽게 이해 할 수 있었음.
추후 강의를 통해 더욱 고도화된 연합학습을 실습을 통해 하고 싶음.



##추가실험

Cifar-10데이터셋을 사용한 코드에서 제시한 기본모델구조 사용에서 더 나아가 비교적 최신에 발표된 EfficientNet을 학습에 사용해보았음.
EfficientNet은 20119년에 발표된 논문으로 현재까지 Convolution 기반의 이미지분류 모델 중 가장 우수한 성능을 보임.
EfficientNet은

1.Network의 Depth
2.Channel width
3.Input Image의 해상도

중 최적을 조합을 찾은 모델 구조이며, 기존 ConvNet 대비 8.4배 감소한 크기에 더불어 6.1배 높은 정확도를 가짐.
따라서 기존 ConvNet구조를 사용하지 않고 Efficient Net을 사용함.


    from efficientnet_pytorch import EfficientNet
    
본 실험에서는 Pretrained 된 EfficientNet을 사용하기 위해 위와 같은 import 명령어를 사용하였음.

main 함수는 다음과 같이 수정함

    def main():
    DEVICE = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")
    print("Centralized PyTorch training")
    print("Load data")
    trainloader, testloader, _ = load_data()
    print("Start training")
    model = EfficientNet.from_pretrained('efficientnet-b0', num_classes=10)
    net = model.to(DEVICE)
    train(net=net, trainloader=trainloader, epochs=2, device=DEVICE)
    print("Evaluate model")
    loss, accuracy = test(net=net, testloader=testloader, device=DEVICE)
    print("Loss: ", loss)
    print("Accuracy: ", accuracy)

client의 epoch는 2, Server의 round는 3으로 설정후 실험한 결과는 다음과 같음.

![image](https://user-images.githubusercontent.com/64252911/192434644-e2392d8e-999f-4eed-8446-bc3db74ae020.png)

Pretrained model을 사용하였기 때문에 실험결과가 기존모델에서 0.02%에서 0.68%까지 상당한 발전을 보임.
