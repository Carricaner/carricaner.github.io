# 進入設計模式的世界 -- 狀態機模式 ( State pattern )


<!--more-->

## 前言 <a id="preface"></a>

「狀態機模式」( State Pattern ) 是眾多設計模式中的一種，主要是用來處理一種物件可能依照不同的狀態而有不同行為的情況。

可以應用的場景有錄音機、紅綠燈等。若是商業方面的應用話，如包裹運送、文件傳簽等。

本文就以公司裡面會出現的「傳簽」來當作狀態模式的應用場景。

<br>
{{< figure src="images/unplash-signature.jpeg" caption="*Photo by Signature Pro in Unsplash*">}}
<br>

## 本文 <a id="main-content"></a>

### 場景一：簡單的傳簽系統<a id="scenario-1"></a>

假設現在有一間新創公司剛成立一年，內部設計出如下圖的傳簽流程：

{{< mermaid >}}
graph LR;
    A([Draft]) -->|Send to Boss| B([Waiting for Boss])
    B -->|Boss signed| C([Approved])
{{< /mermaid >}}

我準備了以下幾項東西來符合需求：
- 狀態機 ( State Machine ) 

- 狀態 ( State )

- 狀態資訊 ( StateInfo )

<br>

```java
public interface StateMachine {
    /* Switch to next state */
    void nextState();

    /* Print current information */
    void info();
}
```
```java
public interface State {
    /* Set next state */
    void setNextState(State nextState);

    /* Switch to & return next state */
    State nextState();

    /* Print current state information */
    void printStateInfo();
}
```
{{< highlight java >}}
public class SignatureStateMachine implements StateMachine {

    private State currentState;

    public SignatureStateMachine(State currentState) {
        this.currentState = currentState;
    }

    @Override
    public void nextState() {
        if (currentState != null) currentState = currentState.nextState();
    }

    @Override
    public void info() {
        if (currentState != null) {
            currentState.printStateInfo();
        } else {
            System.out.println("Already approved!");
        }
    }
}
{{< / highlight >}}
{{< highlight java >}}
public class SignatureState implements State {

    private State nextState;

    private StateInfo info;

    public SignatureState(StateInfo info) {
        this.info = info;
    }

    @Override
    public void setNextState(State nextState) {
        this.nextState = nextState;
    }

    @Override
    public State nextState() {
        return nextState;
    }

    @Override
    public void printStateInfo() {
        System.out.println("Now, the state is " + info.getInfo());
    }
}
{{< / highlight >}}
```java
public enum StateInfo {
    DRAFT("Draft"),
    WATING_FOR_BOSS("Waiting for boss"),
    APPROVED("Approved");

    private String info;
    StateInfo(String info) {
        this.info = info;
    }

    public String getInfo() {
        return info;
    }
}
```

<br>

接著，我們來操作這次的狀態轉移。

```java
public class SignatureDemo {
    public static void main(String[] args) {
        State draftState = new SignatureState(StateInfo.DRAFT);
        State waitingForBossState = new SignatureState(StateInfo.WATING_FOR_BOSS);
        State approvedState = new SignatureState(StateInfo.APPROVED);

        draftState.setNextState(waitingForBossState);
        waitingForBossState.setNextState(approvedState);
        approvedState.setNextState(null);

        StateMachine signSystem = new SignatureStateMachine(draftState);
        signSystem.info(); // "Now, the state is Draft"

        signSystem.nextState();
        signSystem.info(); // "Now, the state is Waiting for boss"

        signSystem.nextState();
        signSystem.info(); // "Now, the state is Approved"

        signSystem.nextState();
        signSystem.info(); // "Already approved!"

        signSystem.nextState();
        signSystem.info(); // "Already approved!"
    }
}

```

<br>

### 場景二：漸漸複雜的傳簽系統<a id="scenario-2"></a>

上個例子比較簡單，現在開始加入一些現實上可能會遇到的問題，並且調整我們的程式碼來符合需求。加入考慮的項目如下：

1. 公司雇用一位部門主管，文件簽核必須依序簽合。

2. 部門主管與老闆可以退回文件，文件傳回上一狀態。

<br>

新的傳簽流程：

{{< mermaid >}}
graph LR;
    A([Draft]) -->|Send to Leader| B([Waiting for Leader])
    B -.->|Leader rejects| A
    B -->|Send to Boss| C([Waiting for Boss])
    C -.->|Boss rejects| B
    C -->|Boss signed| D([Approved])
{{< /mermaid >}}

<br>

這次的情境不再是只有單一的狀態轉移，因此我們新建立簽署動作 ( Sign action ) 來當作狀態轉移的判斷標準。

簽署動作 ( Sign action ):
```java
public enum SignAction {
    SIGN(1),
    REJECT(2);

    private Integer code;

    SignAction(Integer code) {
        this.code = code;
    }
}
```

我們沿用上面的狀態機 ( StateMachine ) 介面，但是改寫狀態 ( State ) 介面，並且重新實作它們。

重新定義的狀態介面如下：
```java
public interface State {
    /* Register a next state */
    void registerNextState(SignAction action, State nextState);

    /* Switch to & return next state */
    State nextState(SignAction action);

    /* Print current state information */
    void printStateInfo();
}
```
<br>

狀態機 ( State Machine )

```java
public class SignatureStateMachine implements StateMachine {

    private State currentState;

    public SignatureStateMachine(State currentState) {
        this.currentState = currentState;
    }

    @Override
    public void nextState(SignAction action) {
        if (currentState != null) currentState = currentState.nextState(action);
    }

    @Override
    public void info() {
        if (currentState != null) {
            currentState.printStateInfo();
        } else {
            System.out.println("Already approved!");
        }
    }

    /* The static method to generate a custom signature system */
    public static SignatureStateMachine generateSignatureSystem() {
        State draftState = new SignatureState(StateInfo.DRAFT);
        State waitingForLeaderState = new SignatureState(StateInfo.WATING_FOR_LEADER);
        State waitingForBossState = new SignatureState(StateInfo.WATING_FOR_BOSS);
        State approvedState = new SignatureState(StateInfo.APPROVED);

        draftState.registerNextState(SignAction.SIGN, waitingForLeaderState);
        waitingForLeaderState.registerNextState(SignAction.SIGN, waitingForBossState);
        waitingForLeaderState.registerNextState(SignAction.REJECT, draftState);
        waitingForBossState.registerNextState(SignAction.SIGN, approvedState);
        waitingForBossState.registerNextState(SignAction.REJECT, waitingForLeaderState);

        return new SignatureStateMachine(draftState);
    }
}
```

狀態 ( State )
```java
public class SignatureState implements State{

    private Map<SignAction, State> map = new EnumMap<>(SignAction.class);

    private StateInfo info;

    public SignatureState(StateInfo info) {
        this.info = info;
    }

    @Override
    public void registerNextState(SignAction action, State nextState) {
        map.put(action, nextState);
    }

    @Override
    public State nextState(SignAction action) {
        if (map.containsKey(action)) {
            return map.get(action);
        } else {
            throw new StateError.NoSuchStateException();
        }
    }

    @Override
    public void printStateInfo() {
        System.out.println("Now, the state is " + info.getInfo() + ".");
    }
}
```


<br>

此外，我們把狀態機直接封裝在文件 ( File ) 內部，讓操作的人可以只需針對文件進行操作，不用知道內部的狀態實際上是怎麼運作。

<br>

文件 ( File )
```java
public class File {
    /* The file's content */
    private String content;

    /* Whether the file is submitted. */
    private boolean submitted = false;

    /* The map to store role & password pairs. */
    private Map<String, String> roleToTokenMap = Map.ofEntries(
            Map.entry("Leader", "leader-pass-123"),
            Map.entry("Boss", "boss-pass-456")
    );

    /* Inner state machine */
    private SignatureStateMachine stateMachine = SignatureStateMachine.generateSignatureSystem();

    public File(String content) {
        this.content = content;
    }

    /* Display the file's state information */
    public void stateInfo() {
        stateMachine.info();
    }

    /* To sign the file */
    public void sign(String role, String password, SignAction action) {
        if (!submitted) throw new StateError.NotSubmittedException();

        if (roleToTokenMap.containsKey(role)
                && roleToTokenMap.get(role).equals(password)) {
            stateMachine.nextState(action);

            if (role.equals("Leader") && SignAction.REJECT.equals(action)) {
                submitted = false;
            }
        }
    }

    /* To submit the file */
    public void submit() {
        if (!submitted) {
            stateMachine.nextState(SignAction.SIGN);
            submitted = true;
        } else {
            throw new StateError.AlreadySubmittedException();
        }
    }
}
```

<br>

狀態轉移如下：
```java
public class SignatureDemo2 {
    public static void main(String[] args) {
        String fileContent = "Promotion request.";
        File newFile = new File(fileContent);
        newFile.stateInfo(); // "Now, the state is Draft."

        newFile.submit();
        newFile.stateInfo(); // "Now, the state is Waiting for leader."

        newFile.sign("Leader", "leader-pass-123", SignAction.SIGN);
        newFile.stateInfo(); // "Now, the state is Waiting for boss."

        newFile.sign("Boss", "boss-pass-456", SignAction.REJECT);
        newFile.stateInfo(); // "Now, the state is Waiting for leader."

        newFile.sign("Leader", "leader-pass-123", SignAction.SIGN);
        newFile.stateInfo(); // "Now, the state is Waiting for boss."

        newFile.sign("Boss", "boss-pass-456", SignAction.SIGN);
        newFile.stateInfo(); // Now, the state is Approved.
    }
}
```

<br>

### 更多複雜的場景 <a id="scenario-future"></a>

真正在處理公司內部的業務邏輯的時候，複雜度往往沒有像上面兩個例子一樣簡單，好比下面傳簽的場景：

{{< mermaid >}}
flowchart LR
    A([Draft]) --> RD-Hsinchu
    subgraph RD-Hsinchu
    C([Senior RD 1]) --> D([Senior RD 2])
    D --> E([RD Manager])
    end
    RD-Hsinchu --> PIE-Tainan
    subgraph PIE-Tainan
    F([Senior PIE]) --> G([RE Manager])
    end
PIE-Tainan --> K([Boss])
    K --> R([Approved])

{{< /mermaid >}}

在處理複雜場景的時候，還可以考慮以下幾點，讓狀態模式更動態 ( 當然也就更複雜... :sweat_smile: )

- 抽象化轉換過程 ( Transition )。

- 封裝狀態機 ( State Machine ) 到會改變狀態的物件當中。

- 使用列舉 ( Enumerate ) 方式來定義狀態資訊 ( State Information )、轉換過程 ( Transition ) 等。

- 除了可以轉換的狀態之外，也紀錄上一個狀態。

- 有機會轉換失敗的話，加入回滾 ( Rollback )的轉換。

<br>

當然...這篇文章就不寫出如何處理這樣的場景啦! :wink:

<br> 

## 總結
- 狀態模式 ( State Machine ) 主要是用來處理一種物件可能依照不同的狀態而有不同行為的情況。

- 狀態模式可以有不同的實作方式，但是核心精神在於處理狀態與轉移之間的關係。

- 設計過程中，可能涉及[有限狀態機 ( Finite-State Machine )](https://zh.wikipedia.org/zh-tw/%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E6%9C%BA) 模型。

- 常見的生活例子有紅綠燈、各式電器等等。商業應用的話好比傳簽系統、包裹配送系統、電子下單系統等等。

- 若面對比較複雜的場景，Java 的使用者可以選擇套用 [Spring Statemachine](https://spring.io/projects/spring-statemachine) 。

<br>

## 參考
- [State Machines: Components, Representations, Applications](https://www.baeldung.com/cs/state-machines)

- [State Design Pattern in Java](https://www.baeldung.com/java-state-design-pattern)

- [Validating Input with Finite Automata in Java](https://www.baeldung.com/java-finite-automata)

- [A Guide to the Spring State Machine Project](https://www.baeldung.com/spring-state-machine)

- [Implementing Simple State Machines with Java Enums](https://www.baeldung.com/java-enum-simple-state-machine)

- [Simple State Machines in Java](https://balaaagi.medium.com/simple-state-machines-in-java-ed2a3db35a34)
