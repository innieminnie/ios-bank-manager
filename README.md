# Bank Manager

---

비동기적으로 은행 업무를 처리하는 과정을 콘솔 앱으로 구현

- 프로젝트 진행 기간: 2021.01.4 ~ 2021.01.15
- 팀 프로젝트: 이니, 오동나무

# 구현한 기능

---

### Step 1

![console01.png](/image/console01.png)

- 콘솔 창에서 유저의 입력을 받도록 구현
- 한 번에 1명의 고객을 처리할 수 있는 은행원 1명이 업무를 처리하도록 구현
- 업무 마감 시점에 처리한 고객수와 총 업무 시간을 계산하여 출력하도록 구현
- 고객은 10~30 명이 랜덤으로 방문합니다.

### Step 2

![console02.png](/image/console02.png)

- 높은 우선순위 등급(VVIP/VIP/일반)을 가진 고객이 먼저 처리될 수 있도록 배열을 재정렬하는 기능 구현
- 고객의 업무(대출/예금)에 따라 소요되는 시간이 달라지도록 구현
- 한 번에 1명의 고객을 처리할 수 있는 은행원 3명이 업무를 처리하도록 구현

### Step 3

![console03.png](/image/console03.png)

- 대출업무 처리 과정에 대출심사 과정을 추가
- Singleton 패턴을 이용하여 대출심사를 진행하는 본사 객체 구현
- 은행원이 대출업무 고객을 본사에 승인 요청 보내고 응답을 기다린 후 업무가 처리될 수 있도록 구현
- 본사는 한 번에 1명의 고객에 대한 승인요청을 처리할 수 있음

# 주요 고민 포인트

---

## 🤔 콘솔 출력 문자열 관리

 출력 문자열을 관리하는 열거형을 만드는 시도를 해보았습니다. 출력문이 많고 분산되어 있는 프로젝트의 경우 코드가 많아질수록 원하는 문자열을 찾기 힘들어질 것 같다는 생각이 들었습니다. 유지, 보수 측면에서 관리를 용이하게 해보고자 시작했으나 이러한 경우 몇가지 장점과 단점이 있었습니다.

```swift
enum ProcessStatus: String {
    case start = "업무 시작"
    case done = "업무 종료"
    case startExamination = "심사 시작"
    case completeExamination = "심사 완료"
    case failExamination = "심사 실패"
}

enum ConsoleOutput {
    case invalidInput
    case menuChoice
    case bankOpening
    case bankClosing(Bank)
    case currentProcess(ClientOperation, ProcessStatus)

    var message: String {
        switch self {
        case .invalidInput:
            return "잘못된 입력값입니다. 다시 시도해주세요."
        case .menuChoice:
            return " 1 : 은행개점 \\n 2 : 종료\\n 입력 : "
        case .bankOpening:
            return "은행개점"
        case let .bankClosing(bank):
            return "업무가 마감되었습니다. 오늘 업무를 처리한 고객은 총 \\(bank.totalClientCount)명이며, 총 업무시간은 \\(bank.totalOperateTime)초입니다."
        case let .currentProcess(client, processStatus):
            guard let clientWaitingNumber = client.waitingNumber, let clientGrade = client.grade?.description, let  clientBusiness = client.business?.rawValue else {
                return BankOperationError.unknownError.rawValue
            }

            return "\\(clientWaitingNumber)번 \\(clientGrade)고객 \\(clientBusiness)\\(processStatus.rawValue)"
        }
    }
}

```

- 장점: 한 곳에서 문자열을 관리하고 있기 때문에, 만약 전체적인 콘솔 출력문이 수정될 경우에 유리하다. 수정을 위해 코드 구석구석을 찾아다닐 필요가 없다.
- 단점: 응집도가 낮아지고 결합도가 높아졌다. 코드와 관련된 출력 문자열이 다른 객체에 있기 때문에 관련된 정보가 흩어지는 느낌이 있다. 또한 새로운 객체가 생성되고 서로 소통해야하는 상황이 되므로 결합도가 높아졌다고 볼 수 있다.

 이 프로젝트에서는 전반적으로 객체지향, 응집도와 결합도에 대한 고민을 많이 했었습니다. 출력 문자열을 관리하는 것도 응집도와 결합도에 대한 파생된 고민이었습니다. 깔끔하게 문제가 해결된 것 같지는 않습니다만 새로운 관점을 얻을 수 있는 경험이었습니다.

## 🤔 여러 명이 동시에 업무 처리하기, Dispatch vs Operation

여러 명의 은행원이 업무를 처리하는 과정을 구현하기 위해서는 동시성 프로그래밍, 동기/비동기에 대한 이해가 필요했습니다. 단번에 이해하기 쉽지 않았지만 여러 코드를 작성해보고 팀원, 캠퍼들과 논의하면서 이해할 수 있었습니다. 

### ✐ Step1

```swift
class BankClerk {
    let counterNumber: Int
    var totalWorkingTime: Float = 0
    var totalProcessedClients: Int = 0
    var workingStatus: BankClerkStatus {
        didSet {
            if workingStatus == .workable {
                NotificationCenter.default.post(name: Notification.Name("workable"), object: nil, userInfo: ["counterNumber": self.counterNumber])
            }
        }
    }

	...
}

class Bank {
	...

    init(employeeNumber: Int) throws {
        self.serviceCounter = try loadBankClerks(of: employeeNumber)
        NotificationCenter.default.addObserver(self, selector: #selector(assignClient), name: NSNotification.Name("workable"), object: nil)
    }

	...

	@objc private func assignClient(_ noti: Notification) { ... }

	func makeAllClerksWorkable() {
        for clerk in serviceCounter.values {
            clerk.workingStatus = .workable
        }
    }

	...
}

```

- 동시성 프로그래밍에 대한 개념이 잡히지 않았을 때 작성했던 방법입니다. 객체 간 결합도를 낮출 수 있는 방법으로 고민하여 Notification을 사용해보았습니다.
- 은행원은 업무 시작, 종료 시점에 workingStatus 프로퍼티가 변경됩니다.
- didSet을 이용하여 은행원의 업무가 끝나면 다음 손님을 받기위해 Notification으로 상태를 알려주고 이어서 업무를 처리하도록 구현해보았습니다.
- 하지만 2명 이상의 은행원이 일을 하게 될 경우, 먼저 배치된 은행원의 didSet이 연속적으로 처리되어`makeAllClerkWorkable()` 메서드의 for 문이 의미가 없는, 먼저 배치된 은행원만이 일을 하는 문제가 발생하였습니다.

### ✐ Step2

 step1에서의 문제를 해결하기 위해서는 **비동기** 작업을 구현해주어야 했습니다. Step2에서는 DispatchQueue를 사용하여 구현해보았습니다.

```swift
class Bank {
    ...

    func makeAllClerksWork() {
        let semaphore = DispatchSemaphore(value: 1)
        let counterGroup = DispatchGroup()

        for i in 1...clerkNumber {
            let dispatchQueue = DispatchQueue(label: "Counter\\(i)Queue")

            dispatchQueue.async(group: counterGroup) {
                self.handleWaitingList(with: semaphore)
            }
        }

        counterGroup.wait()
    }

	private func handleWaitingList(with semaphore: DispatchSemaphore) {
        let bankClerk = BankClerk()

        while !self.waitingList.isEmpty {
            semaphore.wait()
            guard let client = self.waitingList.first else {
                return
            }
            self.waitingList.removeFirst()
            semaphore.signal()

            bankClerk.handleClientBusiness(of: client)
            self.totalProcessedClientsCount += 1
        }
    }

	...
}

```

- GCD를 이용하여 비동기적으로 업무를 처리해주니 step1에서의 문제를 해결할 수 있었습니다.
- 이 과정에서는 race condition을 경험하게 되었고 DispatchSemaphor를 이용하여 배열에 여러 thread가 접근하는 문제를 해결할 수 있었습니다.
- 은행업무는 비동기적으로 처리되기 때문에 아래와 같이 업무가 시작되자마자 마감 멘트가 출력되는 문제가 있었는데 이는 DispatchGroup을 사용하여 모든 은행원의 업무가 종료될 때까지 `counterGroup.wait()` 해주어서 해결할 수 있었습니다.

![console04.png](/image/console04.png)

### ✐ Step3

 Operation에 대해 알아보고 이번 프로젝트에서는 GCD보다 Operation이 더 적합할 것 같다는 생각을 하게 되어서 Operaion을 사용하여 코드를 수정해보았습니다.

```swift
class Bank {
    ...

    func startWorking() {
        let clientOperationQueue = OperationQueue()
        clientOperationQueue.maxConcurrentOperationCount = clerkNumber
        clientOperationQueue.addOperations(waitingList, waitUntilFinished: true)
    }

	...
}

class ClientOperation: Operation {
    private(set) var waitingNumber: Int?
    private(set) var grade: ClientGrade?
    private(set) var isLoanQualified: Bool?
    private(set) var business: BusinessType?

    init(waitingNumber: Int) { ... }

    override func main() { ... }

    private func operateBusiness(of client: ClientOperation) throws { ... }

    private func handleDepositBusiness() { ... }

    private func handleLoanBusiness(of client: ClientOperation) throws { ... }
}

```

- ClientOperation 객체 내부에서 Client와 관련된 모든 처리를 해줄 수 있었습니다.
- OperationQueue를 사용하여 순서대로 업무가 처리되는 queue가 만들어지므로 race condition에 대해서 따로 처리해주지 않아도 됩니다.
- 객체의 역할이 보다 명확하게 나누어져 코드를 읽고 관리하기 쉬워졌습니다.

# 프로젝트 회고

---

- 동시성 프로그래밍에 대해 깊게 고민해볼 수 있었습니다. 또한 전체적인 수정이 많았던 프로젝트였지만 과감하게 여러 시도를 해볼 수 있어서 좋았습니다.
- 프로젝트 초반에는 협업, 페어 프로그래밍에 미숙해서 팀원의 코드와 제 코드를 합치는데에 많은 시간과 체력이 소모되었습니다. 이후 협업이 어떻게 이루어지는지에 대해 자문을 구해보았고, 이번 프로젝트에서는 페어 프로그래밍을 집중적으로 경험해보게 되었습니다. 나의 코드를 설명하고 팀원의 코드를 읽는 과정에서 코드가 더욱 단단해졌고 다양한 시도를 할 수 있었습니다.
- 처음 GCD와 Operation을 공부하기 전에 Operation이 더 복잡해서 간단한 처리는 대부분 GCD를 이용한다는 글들을 많이 접하게 되었습니다. 어떤 것이 프로젝트에 더 적합할지 비교해보지 않고 GCD로 덥석 구현했던 점이 아쉽게 느껴집니다.