---
title: Minecraft Core Shader로 만드는 애니메이션 API
description: 모드 없이 엔티티를 애니메이팅하는 API 구축
keywords:
  - Minecraft
  - Resourcepack
tags:
  - Shader
  - GLSL
  - Resourcepack
categories:
  - Minecraft
  - Dev
date: 2024-07-16T15:37:00+09:00
draft: true
isCJKLanguage: true
aliases: 
weight: 10
build:
  list: always
  publishResources: true
  render: always
---
마인크래프트 스냅샷 21w10a에는 Lush Cave를 비롯한 몇몇 기능들이 소개되었으나 그 진가는 아래의 작은 *기술적 변경사항*이었다. OpenGL 버전이 오른 것뿐만 아니라, Core Shader라는 시스템이 추가된 것이다.

이번 포스트는 이 Core Shader를 응용해 