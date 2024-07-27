---
title: ScreenBox 기반 만들기
description: ScreenBox 프로젝트의 토대를 세우자
keywords:
  - Java
  - Windows
tags:
  - Kotlin
  - Windows
categories:
  - Dev
date: 2024-07-28T01:18:04+09:00
draft: false
isCJKLanguage: true
aliases: 
weight: 10
build:
  list: always
  publishResources: true
  render: always
---
## Introduction
Dynamited 작업 중 알 수 없는 버그가 계속 발생하다보니 뭔가 흥미가 떨어졌다...

별로 좋은 방법은 아닐 것 같지만, '다른 성격의 프로젝트를 병행하면 한 쪽이 질릴 때 나머지 하나를 할 수 있지 않을까?' 라는 생각으로 새로운 프로젝트를 시작한다.

이름하여 ScreenBox. 10초만에 지은 거라 그런지 좀 별로지만 넘어가자. 이 프로젝트는 데스크탑의 스크린에 이것저것을 표시하는 것을 기반으로 한다. 

정말로 '이것저것'이다. CPU나 램 사용량을 모니터링할 수도 있고, 또는 [Alan Becker](https://www.youtube.com/channel/UCbKWv2x9t6u8yZoB3KcPtnw)의 영상처럼 스틱맨들이 돌아다니게 할 수도 있는 프로그램을 만들고자 한다.

기본적으로 생각하고 있는 것은 위 예시 중 후자이다. 영상처럼은 아니고, [Desktop Goose](https://samperson.itch.io/desktop-goose)의 느낌에 더 가깝다. 이 Desktop Goose와 같은 골자를 공유한다고 생각해도 좋다.

## 프로그래밍 기술
이 프로그램은 특별히 큰 컴퓨팅 성능을 요구하지는 않는다.

데스크탑 화면 맨 위에 창을 만들고 1초에 60번 그래픽을 업데이트할 정도면 된다. 유튜브 영상 재생보다도 싼 프로그램인 셈이다.

또한 고급 프레임워크도 필요 없다. 겨우 창 하나 띄우고 그림 그리는 데 무거운 프레임워크를 사용해야 할까...? UI 작업도 할 생각이 없으니 프레임워크의 중요성이 크게 감소한다.

따라서 Kotlin 언어에서 Swing을 사용하기로 했다. Swing은 JDK에 포함된 GUI 라이브러리인데, 이것으로 고른 이유는 그저 간단해보이기 때문이다. 실제로, 처음 사용하는데도 5분만에 테스트 페이지정도는 만드는 데 성공했다.

Kotlin 언어를 사용하는 이유는... 익숙하기도 했고, 크로스플랫폼에 대한 생각도 있었다. Windows 환경만 생각한다면야 C#으로 WinForms를 사용하겠지만, 기왕이면 Mac이나 Linux계열도 호환되면 좋으니까.

## 작동 코드
사실 코드는 정말 단순해서 자잘한 오류 고치는 시간이 더 걸렸다. 

```kotlin
package kr.lanthanide  
  
import com.sun.jna.Native  
import com.sun.jna.platform.win32.User32  
import com.sun.jna.platform.win32.WinDef  
import com.sun.jna.platform.win32.WinUser  
import java.awt.Color  
import java.awt.Component  
import java.awt.Dimension  
import java.awt.image.BufferedImage  
import java.io.File  
import javax.imageio.ImageIO  
import javax.swing.*  
  
  
fun main() {  
    println("Booting ScreenBox")  
  
    val image: BufferedImage = ImageIO.read(File("src/main/resources/avatar.png"))  
  
    val label = JLabel(ImageIcon(image))  
  
    val pane = JPanel()  
  
    pane.add(label)  
  
    pane.background = Color(0, 0, 0, 0)  
    pane.isOpaque = false  
    pane.size = Dimension(300, 300)  
    pane.maximumSize = Dimension(300, 300)  
    pane.border = BorderFactory.createLineBorder(Color.red)  
    pane.isOpaque = false  
  
    val frame = JFrame("title!")  
    frame.contentPane.add(pane)  
    frame.defaultCloseOperation = JFrame.EXIT_ON_CLOSE  
    frame.extendedState = JFrame.MAXIMIZED_BOTH  
    frame.isUndecorated = true  
    frame.background = Color(0, 0, 0, 0)  
  
    frame.isAlwaysOnTop = true  
    frame.isVisible = true  
  
  
//    setTransparent(frame)  
}  
  
private fun setTransparent(w: Component) {  
    val hwnd: WinDef.HWND = getHWND(w)  
    var wl: Int = User32.INSTANCE.GetWindowLong(hwnd, WinUser.GWL_EXSTYLE)  
    wl = wl or WinUser.WS_EX_LAYERED or WinUser.WS_EX_TRANSPARENT  
    User32.INSTANCE.SetWindowLong(hwnd, WinUser.GWL_EXSTYLE, wl)  
}  
  
/**  
 * Get the window handle from the OS
 */
private fun getHWND(w: Component): WinDef.HWND {  
    val hwnd: WinDef.HWND = WinDef.HWND()  
    hwnd.pointer = Native.getComponentPointer(w)  
    return hwnd  
}
```

이것이 전문이다! 심지어 테스트용으로 마구 붙여놓은 군더더기 코드도 많다. 예를 들어, 저기 `pane.isOpaque` 를 설정하는 부분이 두 번 있다!

간단하게 코드를 설명하자면, 배경이 투명한 패널을 만들고 또 배경이 투명한 프레임에 넣는 것이다. 이 때 패널에는 빨간 테두리와 테스트용 이미지를 넣고, 프레임은 다른 창 위에 표시하며 최대화 상태로 둔다.

Windows라서 이렇게 작동하는지는 모르겠지만, 투명한 곳은 클릭해도 뒤에 있는 창을 클릭한 판정이 나는 것을 이용한 것이다.

참고로 아래 `setTransparent` 를 설정하면 투명하지 않은 부분을 클릭해도 뒤에 있는 창을 클릭한 판정이 된다. 혹시 쓸모가 있을까 싶어 가져왔는데, 아마 쓸 일 없지 않을까 싶다.

아래는 위 프로그램의 실행 화면이다. IDE화면이 비쳐보이는 것을 알 수 있다. 

![테스트 중의 스크린샷](https://github.com/lantice3720/lantice3720.github.io/blob/hugo_port/assets/images/screenbox_1.png?raw=true)