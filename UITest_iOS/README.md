# 使用XCTest撰寫自動化測試&一些測試技巧

## 前言

XCTest為Xcode內建的UI測試工具，無須安裝任何第三方套件(Pod,Swift Package)即可使用。

## 專案設置

## 測試架構與流程

一個測試專案底下可以有多個測試群組(XCTestCase)，而一個測試群組內可以有多個測試細項(Function)，以下依序作介紹。

### 測試架構

測試群組的程式架構如下：
```
clase Group1_FirstTestGroup: XCTestCase {
    override clase func setup(){
        //群組測試開始
    }

    override func setUpWithError() throws {
        //細項測試開始
    }

    override func tearDownWithError() throws {
        //細項測試結束
    }

    override class func tearDown(){
        //群組測試結束
    }

    //撰寫測試細項(function)
    func test1_Function() throws{
        //測試細項1
    }
}
```
說明:
1. 群組名稱：即「物件名稱」，以上述程式碼為例，群組名稱為「`Group1_FirstTestGroup`」，群組的測試順序由此名稱決定
1. 測試細項：其Function名稱必須以「`test`」為開頭，測試順序也是由此名稱決定

### 測試流程

測試流程圖如下:

![UI測試流程](UITest_progress.png)

- SetUp、TearDown(Class Function)：用於整個群組測試的開始與結束，可在此執行群組測試中僅須執行一次的行為，例如：測試所需的資料讀取(SetUp)與儲存(TearDown)。
- SetUpWithError、TearDownWithError：於每個測試細項執行前後執行，可在此執行重複性的操作，例如：每次測試前須回到某畫面或測試完成後回到首頁。
- Test Function：測試的所有操作細節，依照目的的不同撰寫不同的功能，可依照想測試的畫面中，元件的數量來做規劃，基本的思維邏輯是`UI事件觸發後，確認UI是否符合預期的改變`。

## 開發技巧

依照開我的發經驗分為三類整理：模擬使用者操作([UI操作](#UI操作))、確認操作後的畫面([UI讀取](#UI讀取))、重複性程式與錯誤處裡([其他](#其他))。

<h3 id = "UI操作">UI操作</h3>

1. 開啟APP

    測試的首要步驟就是開啟要測試的APP，也可以將APP退到背景與喚醒，除此之外，部分APP需要進入手機的設定頁面調整WiFi或藍芽也是使用相同的方法，透過輸入Bundle Id來開啟指定的APP，而大部分的系統提示無法被專案讀取到，也是透過特定的Bundle ID來讀取。
    ```
    //開啟目標專案的APP
    XCUIApplication().launch()

    //將APP退到背景
    XCUIApplication().terminate()

    //將APP從背景喚醒
    XCUIApplication().activate()

    //開啟手機的設定
    XCUIApplication(bundleIdentifer:"com.apple.Preferences").launch()

    //讀取手機狀態列(訊號強度、手機電量、網路訊號)與系統提示
    XCUIApplication(bundleIdentifer:"com.apple.springboard")
    ```
1. 點擊操作(tap)

    這是最常見的一種UI操作方式，無論是按鈕、開關、Cell等，大部分的UI元件(`XCUIElement`)都適用，而點擊操作並非只能辦到單純的點擊，也可以依照點擊的次數與手指的數量來達成複雜的操作模式，以下用專案APP中讀取到的第一個按鈕為例。
    ```
    let btn = XCUIApplication().buttons.firstMatch
    
    //點一下
    btn.tap()

    //點兩下
    btn.doubleTap()

    //兩指點一下
    btn.twoFingerTap()

    //三指點五下(可依照需求修改)
    btn.tap(withNumberOfTaps:5, numberOfTouches:3)
    ```

1. 長按與拖曳(press)

    長按與拖曳較為相似，多半用在顯示更多功能或移動元件，以下用table view的第一個cell為例。
    ```
    let firstCell = XCUIApplication().tables.cells.firstMatch

    //長按五秒
    firstCell.press(forDuration:5)

    //長按三秒後拖曳至最後一個Cell
    if let lastCell = XCUIApplication().tables.cells.allElementsBoundByIndex.last{
        firstCell.press(forDuration:3, thenDragTo:lastCell)
    }
    
    //長按兩秒後快速拖曳至最後一個Cell並懸停一秒
    if let lastCell = XCUIApplication().tables.cells.allElementsBoundByIndex.last{
        firstCell.press(forDuration:2, thenDragTo:lastCell, withVelocity:.fast, thenHoldForDuration:1)
    }
    ```
1. 滑動(swipe)

    上下左右滑動，皆可指定滑動的速度，以下用Collection View為例。

    ```
    let collectionView = XCUIApplication().collectionViews.firstMatch

    //上滑
    collectionView.swipeUp()

    //下滑
    collectionView.swipeDown()

    //快速左滑
    collectionView.swipeLeft(velocity: .fast)

    //慢速右滑
    collectionView.swipeRight(velocity: .slow)
    ```

1. 縮放(zoom)與旋轉(rotate)

    

<h3 id = "UI讀取">UI讀取</h3>

<h3 id = "其他">其他</h3>

## 測試計畫

## 附註