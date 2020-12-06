---
layout: article

title:  破解最新抓包工具 Charles
date:   2020-12-06 17:25:00 +0800

tags: reverse
key: reverse_charles
---

Charles 是常用的抓包工具，使用 Java 语言写的，支持跨平台。目前最新版本是 4.6.1，本文记录下对该版本的破解。

<!--more-->

## 工具

### JD-GUI

反编译 Java 程序，可将 Java class 文件反编译为 Java 源码。[下载地址](https://github.com/java-decompiler/jd-gui).

### Recaf

Java bytecode editor，可对 class 文件进行修改。 [下载地址](https://github.com/Col-E/Recaf).



## 逆向 Charles

使用 JD-GUI 打开 lib 目录下的 charles.jar，注册和校验逻辑都在该 jar 中。

### 寻找关键逻辑

点击菜单 Help --> Register Charles，会弹出下面注册对话框：

　　　　　　　　　　　　![graph]({{"/assets/pictures/reverse/charles/register_dlg.png" | prepend:site.baseurl}})

通过字符串 "Registered Name" 找到上面对话框代码 RegisterFrame：

```java

package com.xk72.charles.gui.frames;
... ...

public class RegisterFrame extends JDialog {
  ... ...
  public RegisterFrame(Frame paramFrame) {
    super(paramFrame, true);
    setTitle("Register Charles");
    this.tName = new JTextField(20); // 输入 Name
    this.tSerial = new JTextField(20); // 输入的 Key
    this.bRegister = new JButton("Register");
    this.bCancel = new JButton("Cancel");
    Container container = getContentPane();
    container.setLayout((LayoutManager)new MigLayout("wrap,fill", "[label][fill,grow]"));
    container.add(new JLabel("Registered Name:"));
    container.add(this.tName);
    container.add(new JLabel("License Key:"));
    container.add(this.tSerial);
    container.add(this.bCancel, "tag cancel,split 2,span,center");
    container.add(this.bRegister, "tag ok");
    this.bCancel.addActionListener(new CHDR(this));
    this.bRegister.addActionListener(new mYlx(this)); // 此处为点击 Register 后的逻辑
    pack();
    if (paramFrame != null) {
      Point point = new Point(paramFrame.getLocation());
      point.translate(20, 20);
      setLocation(point);
    } 
    getRootPane().setDefaultButton(this.bRegister);
    getRootPane().getInputMap(1).put(KeyStroke.getKeyStroke("ESCAPE"), "escape");
    getRootPane().getActionMap().put("escape", new RegisterFrame$3(this));
  }
... ...
}
```

查看点击 Button 后的逻辑：

```java
package com.xk72.charles.gui.frames;
... ...

class mYlx implements ActionListener {
  mYlx(RegisterFrame paramRegisterFrame) {}
  
  public void actionPerformed(ActionEvent paramActionEvent) {
    String str1 = this.XdKP.tName.getText().trim(); 
    String str2 = this.XdKP.tSerial.getText().trim();
    if (str1.length() > 0 && str2.length() > 0) { // str1 和 str2 分别代表之前输入的 Name 和 Key
      String str = MfoV.XdKP(str1, str2);
      if (str != null) {
        ExtendedJOptionPane.XdKP(this.XdKP, str, "Charles Registration", 2);
      } else { // 若调用 MfoV.XdKP 返回为 NULL，则表示成功注册
        ExtendedJOptionPane.XdKP(this.XdKP, "Thank you for registering. Charles will now close. Please start Charles again to continue.", "Charles Registration", 1);
        CharlesContext charlesContext = CharlesContext.getInstance();
        charlesContext.getConfiguration().getRegistrationConfiguration().setName(str1);
        charlesContext.getConfiguration().getRegistrationConfiguration().setKey(str2);
        charlesContext.exit(0, true);
      } 
    } 
  }
}

```

上述可以看到，点击注册后，会调用 MfoV.XdKP 方法，若该方法返回 NULL 表示注册成功，返回非 NULL，则会将该字符串输出，该字符串是注册失败的提示信息。

### 注册校验逻辑

MfoV.XdKP 调用流程：

* MfoV.XdKP(name, key)  --> MfoV.MfoV(name, key) --> MfoV.MfoV(name, key, 4)

```java
  MfoV(String name, String key, int paramInt) {
    try {
      if (!XdKP(name, key, paramInt))
        throw new LicenseException(XdKP(2)); 
    } catch (NumberFormatException numberFormatException) {
      throw new LicenseException(XdKP(1));
    } 
    this.aDeW = name;
    this.DHBI = true;
  }
```

此处调用 XdKP 方法，若返回为 true 则表示注册成功，会将 Name 保存到 aDeW 变量中，将校验结果保存在 DHBI 变量。

XdKP 调用流程：

* MfoV.XdKP --> MfoV.CYm

```java
  private long eCYm(String name, String key, int paramInt) {
    if (key.length() != 18) // key 长度必须是 18
      throw new LicenseException(XdKP(0)); 
    if (key.equalsIgnoreCase("7055ce2f8cb4f9405f") || key.equalsIgnoreCase("5bae9d8cdea32760ae") || key.equalsIgnoreCase("f3264994d9ea6bc595") || key.equalsIgnoreCase("b9930cef009d3a7865") || key.equalsIgnoreCase("62bd6a5f95aa67998e") || key.equalsIgnoreCase("a1c536c35904e64584") || key.equalsIgnoreCase("d6e5590ecc05edd9b3") || key.equalsIgnoreCase("8fbe36ce2726458b18") || key.equalsIgnoreCase("042a8352caf1188945") || key.equalsIgnoreCase("9d26d5088770221c3c") || key.equalsIgnoreCase("e19b2a01905e4129bf") || key.equalsIgnoreCase("68ebe4c9d792f31057") || key.equalsIgnoreCase("4e4beb8a43e9feb9c7") || key.equalsIgnoreCase("d04d85b44b306fc9ec") || key.equalsIgnoreCase("2b5d21a38c9452e342") || key.equalsIgnoreCase("88cb89c26a813bce44") || key.equalsIgnoreCase("76c9ee78c8ab124054") || key.equalsIgnoreCase("729db7c98163ac7d3d") || key.equalsIgnoreCase("7c1d4761993c412472") || key.equalsIgnoreCase("08bc0b7ec91cd0f4aa") || key.equalsIgnoreCase("25bafae175decaedcc") || key.equalsIgnoreCase("3181aae6822ef90ccd") || key.equalsIgnoreCase("d7a8fe9dc9dc919f87") || key.equalsIgnoreCase("728dae81d9d22aca03") || key.equalsIgnoreCase("119a9b593348fa3e74") || key.equalsIgnoreCase("04ab87c8d69667878e") || key.equalsIgnoreCase("4b282d851ebd87a7bb") || key.equalsIgnoreCase("ed526255313b756e42") || key.equalsIgnoreCase("ed5ab211362ab25ca7") || key.equalsIgnoreCase("18f4789a3df48f3b15") || key.equalsIgnoreCase("67549e44b1c8d8d857") || key.equalsIgnoreCase("4593c6c54227c4f17d") || key.equalsIgnoreCase("1c59db29042e7df8ef") || key.equalsIgnoreCase("a647e3dd42ce9b409b") || key.equalsIgnoreCase("7e06d6a70b82858113") || key.equalsIgnoreCase("ef4b5a48595197a373") || key.equalsIgnoreCase("0ac55f6bebd0330640") || key.equalsIgnoreCase("1beda9831c78994f43") || key.equalsIgnoreCase("8a2b9debb15766bff9") || key.equalsIgnoreCase("da0e7561b10d974216") || key.equalsIgnoreCase("86257b04b8c303fd9a") || key.equalsIgnoreCase("a4036b2761c9583fda") || key.equalsIgnoreCase("18e69f6d5bc820d4d3") || key.equalsIgnoreCase("a13746cb3d1c83bca6") || key.equalsIgnoreCase("a4036b2761c9583fda"))
      throw new LicenseException(XdKP(1)); // key 不能是上面这些黑名单
    long l1 = eCYm(key);
    int i = uQqp(key);
    AhDU(-5408575981733630035L);
    long l2 = uQqp(l1);
    if (eCYm(l2) != i)
      throw new LicenseException(XdKP(1)); 
    this.aTLp = (int)(l2 << 32L >>> 32L >>> 24L);
    if (this.aTLp == 1) {
      this.CjEc = LicenseType.XdKP;
    } else if (this.aTLp == paramInt) {
      switch ((int)(l2 << 32L >>> 32L >>> 16L & 0xFFL)) {
        case 1:
          this.CjEc = LicenseType.XdKP;
          break;
        case 2:
          this.CjEc = LicenseType.eCYm;
          break;
        case 3:
          this.CjEc = LicenseType.uQqp;
          break;
        default:
          throw new LicenseException(XdKP(1));
      } 
    } else {
      if (this.aTLp < paramInt)
        throw new LicenseException(XdKP(3)); 
      throw new LicenseException(XdKP(1));
    } 
    AhDU(8800536498351690864L);
    try {
      int j = XdKP(eCYm(name.getBytes("UTF-8")));
      j ^= (int)(l2 >> 32L);
      return 0xA58D19C600000000L | j << 32L >>> 32L;
    } catch (UnsupportedEncodingException unsupportedEncodingException) {
      return -1L;
    } 
  }
```

校验逻辑：
1. key 长度必须是 18；
2. key 不能是上述黑名单；
3. 然后是通过复杂的算法计算 key 和 name，需要满足一定的关系才成立。

### 寻找启动判断注册逻辑

Charles 在未注册情况下启动，会显示下面信息：

　　　　　　　　![graph]({{"/assets/pictures/reverse/charles/splash_screen.png" | prepend:site.baseurl}})

搜索字符串找到代码处：

```java
package com.xk72.charles.gui;
... ...
public class SplashWindow extends JWindow {
    ... ...
    public void showSharewareStatus() { // 显示未注册信息
        showStatus("This is a 30 day trial version. If you continue using Charles you must\npurchase a license. Please see the Help menu for details.");
    }

    public void showRegistrationStatus() {
      if (MfoV.XdKP()) { // 判断是否注册
        showStatus("Registered to: " + MfoV.uQqp());
      } else {
        showSharewareStatus();
      } 
    }

```

showRegistrationStatus 方法会调用 MfoV.XdKP 来判断是否注册，然后分别显示文案信息。查看下 MfoV.XdKP 方法：

```java
package com.xk72.charles;
... ...

public class MfoV {
  ... ...
  private static MfoV rziN;

  public static boolean XdKP() {
    return rziN.AhDU(); // 静态方法返回 MfoV 实例
  }

  protected boolean AhDU() {
    return this.DHBI; // 返回 DHBI 的值，该值是上面注册校验成功后，会赋值成 true
  }

```

可以看到，调用 MfoV.XdKP 实际获取的是 DHBI 的值，该值只有注册校验成功后才会赋值为 true.

### 寻找时间限制逻辑

Charles 未注册情况下只允许使用 30 分钟，如下图：

　　　　　　　　![graph]({{"/assets/pictures/reverse/charles/charles_timeout.png" | prepend:site.baseurl}})

通过搜索字符串找到代码处：

```java
package com.xk72.charles;

class XaRp implements Runnable {
  XaRp(uAtD paramuAtD) {}
  
  public void run() {
    MisW.eCYm(this.XdKP.XdKP);
    this.XdKP.XdKP.error("This unlicensed copy of Charles will only run for 30 minutes. You may restart Charles and use it again.\nCharles has taken many years of development. If you continue to use Charles please support that effort and purchase a license.");
    this.XdKP.XdKP.exit(0, true);
  }
}

```
显示逻辑是在 com.xk72.charles.XaRp 类，搜索调用方：

```java
package com.xk72.charles;
... ...

class uAtD extends TimerTask {
  uAtD(CharlesContext paramCharlesContext) {}
  
  public void run() {
    if (MfoV.XdKP()) // 也是通过 MfoV.XdKP 判断是否注册
      return; 
    try {
      SwingUtilities.invokeAndWait(new XaRp(this));
    } catch (InvocationTargetException invocationTargetException) {
    
    } catch (InterruptedException interruptedException) {}
  }
}
```

可以看到，同样是使用 MfoV.XdKP 判断是否注册。

## 破解

### 正向破解

由上面分析知，key 长度是 18字节，并且每个字节都是 16进制，算法可能直接取出使用，那么可以采用穷举法。但是有16^18种可能，数据非常大，并且我本地进行了测试，用i7电脑多线程跑了好几个小时，没有收获，也间接说明这种方法不可取。

### 侵入式破解

Charles 的程序都在本地上，那么可以采用修改本地程序逻辑来破解。

#### 修改注册判断方法

从上面分析看到，最后都是调用 MfoV 的 AhDU 方法，所以我们可以修改该方法的返回值。使用 Recaf 打开 charles.jar，定位到 AhDU 方法上，然后右键选择 “Edit with assembler”：

![graph]({{"/assets/pictures/reverse/charles/patch_charles.png" | prepend:site.baseurl}})

修改方法让始终返回 true，表示已经注册了：

![graph]({{"/assets/pictures/reverse/charles/patch_true.png" | prepend:site.baseurl}})

修改后保存，然后通过菜单 File --> Export program 导出。

#### 增加对 aDeW 赋值

修改后发现，程序不能正常运行，会出现异常：

![graph]({{"/assets/pictures/reverse/charles/crash.png" | prepend:site.baseurl}})

通过错误信息，找到代码处：

```java
  public static String uQqp() {
    return rziN.PRdh();
  }

  protected String PRdh() {
    switch (CHDR.XdKP[this.CjEc.ordinal()]) { // 此处 this.CjEc 为 NULL
      case 1:
        return this.aDeW;
      case 2:
        return this.aDeW + " - Site License";
      case 3:
        return this.aDeW + " - Multi-Site License";
    } 
    return this.aDeW;
  }
```

从上面 eCYm 方法中看到，会有给 this.CjEc 赋值操作，所以在 AhDU 方法中增加对 this.CjEc 赋值。

![graph]({{"/assets/pictures/reverse/charles/patch_cjec.png" | prepend:site.baseurl}})

此时运行可以看到，已经注册了。

![graph]({{"/assets/pictures/reverse/charles/registered.png" | prepend:site.baseurl}})

#### 修改注册名

默认名字是 Unregistered，若想修改名字怎么办？

```java
package com.xk72.charles;

... ...

public class MfoV {
  ... ...
  
  private String aDeW = "Unregistered";

```

名字是保存在变量 aDeW 中，修改注册名就需要修改 aDeW。

![graph]({{"/assets/pictures/reverse/charles/patch_name.png" | prepend:site.baseurl}})

运行效果：

![graph]({{"/assets/pictures/reverse/charles/final.png" | prepend:site.baseurl}})

可以看到注册名已经修改。

## 总结

从破解 Charles 可以学到逆向和修改 Java 程序。

附上修改后的 [charles.jar]({{"/assets/files/charles.jar" | prepend:site.baseurl}})