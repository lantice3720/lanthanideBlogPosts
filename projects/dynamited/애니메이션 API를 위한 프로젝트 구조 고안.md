---
title: 애니메이션 API를 위한 프로젝트 구조 고안
description: 필요한 기능을 생각하고 구현 방법을 고안하자
keywords:
  - Minecraft
  - Java
tags:
  - Kotlin
categories:
  - Minecraft
  - Dev
date: 2024-07-16T21:28:06+09:00
draft: true
isCJKLanguage: true
aliases: 
weight: 10
build:
  list: always
  publishResources: true
  render: always
---
현재의 프로젝트는 간단히 말하자면 애니메이팅 툴의 기능 대부분을 통째로 끌고 오는 것으로, 쉽지 않은 도전이 될 것이라 생각된다. 따라서 코드가 꼬이지 않도록 미리 기능 구상을 하는 것이 옳다.
## 핵심 기능
- NMS를 사용해 디스플레이 엔티티를 보여줌 -> DynaEntity로 Wrap하여 사용.
- Bone과 Armature Class를 사용해 리깅 지원
- Bone을 통한 Forward, Inverse Kinematics 지원
- 블럭벤치 등의 외부 프로그램에서 Keyframe 기반의 애니메이션 import

