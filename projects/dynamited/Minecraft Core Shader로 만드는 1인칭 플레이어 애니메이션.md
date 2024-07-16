---
title: Minecraft Core Shader로 만드는 1인칭 플레이어 애니메이션
description: 모드 없이 엔티티를 애니메이팅하는 API 구축 시작
keywords:
  - Minecraft
  - Resourcepack
  - Core Shader
tags:
  - Shader
  - GLSL
  - Resourcepack
categories:
  - Minecraft
  - Dev
date: 2024-07-16T15:37:00+09:00
draft: false
isCJKLanguage: true
aliases: 
weight: 10
build:
  list: always
  publishResources: true
  render: always
---
마인크래프트 스냅샷 21w10a에는 Lush Cave를 비롯한 몇몇 기능들이 소개되었으나 그 진가는 아래의 작은 *기술적 변경사항*이었다. OpenGL 버전이 오른 것뿐만 아니라, Core Shader라는 시스템이 추가된 것이다.

이번 글은 이 Core Shader를 응용해 1인칭 애니메이션을 만드는 방법을 다룬다. 마인크래프트 버전은 1.20.6을 사용했다.[^1]

혹시 필자의 코드가 궁금하다면, [깃헙 레포](https://github.com/RuinedTechnologyUnify/TheFirstPerson/tree/01e1daaa5bdb9c89ed9edcc4254059f3d5e0d77d)를 참고하자. 다만 이 포스팅과의 시간 차에서 알 수 있듯 최신 코드는 아니라는 점도 알아두면 좋겠다. 참고로, 이 글에서의 설명들은 그저 이해 정도를 위한 것이라 엄밀하지 않으니 자세히 알고 싶은 방문자는 [Reference](#Reference)의 문헌들을 읽어보길 바란다.

아래 영상은 이 글의 결과물이다.
{{< video "https://cdn.discordapp.com/attachments/1246098845654323210/1254820577961574594/Minecraft__1.20.6_-_Multiplayer_3rd-party_Server_2024-06-25_00-28-54.mp4?ex=66973a7b&is=6695e8fb&hm=e2bc6f8a9825598803f35a7a6f88192c7eee171916758ca7b5856d644a04cc9d&" "1인칭 애니메이션 예시" >}}

## 기본 아이디어
Core Shader의 핵심적인 기능은 마인크래프트의 다양한 렌더타입을 손볼 수 있는 것이다. 예를 들어, `rendertype_solid`는 블럭들의 렌더링을 수정할 수 있는 기능을 제공한다. 따라서 이 쉐이더를 수정하면 블럭의 UV 매핑, vertex 위치 등을 변화시킬 수 있다.

이 프로젝트에서는 리소스팩에 스킨을 하드코딩하지 않고 플레이어 모델을 만드는 것이 중요한 목표이다. 이를 위해 특정 Core Shader를 사용해 플레이어 머리의 UV 매핑을 새로 한다. 그런데 이렇게 된다면, 진짜 플레이어 머리를 못 쓰게 될뿐만 아니라 다른 쪽의 팔, 다리, 몸통과 같은 부분에 대한 지원도 불가능하다.
{{< br >}}이것의 해결에는 재미있는 기술이 사용될 수 있는데, 아이템 디스플레이의 `transformation.translation` nbt를 조작해 vertex의 위치를 매우 멀게 하는 것이다. 이는 디스플레이의 Cull이 vertex 위치가 아니라 엔티티 위치 기반으로 작동하기 때문에 가능한 것으로 보인다. 이 다음 쉐이더 내에서 Position을 조작하면 도로 vertex를 가까이 오게 할 수 있다.

또, 1인칭 애니메이션의 구현 시 FOV나 GUI 크기에 관계없이 항상 같은 위치에 디스플레이가 보여야 하고, View Bobbing은 적용되어야 한다. 이를 위해 ProjMat의 값을 조작할 것이다.

## 시작하기
우선 리소스팩 하나가 필요하다. 아마 이 글을 보는 방문자라면 리소스팩의 기본 구성 정도는 잘 알 것이라 생각하지만, 혹시 잘 모르겠다면 [마인크래프트 위키의 문서](https://minecraft.wiki/w/Resource_pack)를 참고하면서 만들어도 좋다. 우선 평소처럼 `assets/minecraft` 폴더를 만들자. 그 다음, `shaders/core` 폴더를 만들 것이다. `shaders` 자체는 이전에도 post 쉐이더용으로 존재하던 폴더지만, 이젠 Core Shader가 추가되어 더욱 풍성해졌다.

이제 마인크래프트의 원본 코드를 가져올 것이다. [mcmeta](https://github.com/misode/mcmeta) 레포나 본인의 versions 폴더에서 원하는 버전을 찾고, `rendertype_entity_translucent`라는 이름의 vsh, fsh, json 파일을 복사해 본인의 리소스팩 폴더에 넣는다.[^2]

마인크래프트는 위 파일처럼 미리 정해진 이름을 가진 json 파일을 `core` 폴더에서 찾고, 이 json에서 가리키는 vsh와 fsh를 실행하는 식으로 작동하는 듯 하다. 
{{< details title="참고: rendertype_entity_translucent.json">}}
```json
{  
  "vertex": "rendertype_entity_translucent",  
  "fragment": "rendertype_entity_translucent",  
  "samplers": [  
    { "name": "Sampler0" },  
    { "name": "Sampler1" },  
    { "name": "Sampler2" }  
  ],  
  "uniforms": [  
    { "name": "ModelViewMat", "type": "matrix4x4", "count": 16, "values": [ 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0 ] },  
    { "name": "ProjMat", "type": "matrix4x4", "count": 16, "values": [ 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 0.0, 0.0, 1.0 ] },  
    { "name": "ColorModulator", "type": "float", "count": 4, "values": [ 1.0, 1.0, 1.0, 1.0 ] },  
    { "name": "Light0_Direction", "type": "float", "count": 3, "values": [0.0, 0.0, 0.0] },  
    { "name": "Light1_Direction", "type": "float", "count": 3, "values": [0.0, 0.0, 0.0] },  
    { "name": "FogStart", "type": "float", "count": 1, "values": [ 0.0 ] },  
    { "name": "FogEnd", "type": "float", "count": 1, "values": [ 1.0 ] },  
    { "name": "FogColor", "type": "float", "count": 4, "values": [ 0.0, 0.0, 0.0, 0.0 ] },  
    { "name": "FogShape", "type": "int", "count": 1, "values": [ 0 ] }  
  ]  
}
```
{{< /details >}}
{{< details title="참고: rendertype_entity_translucent.vsh" >}}
```glsl
#version 150

#moj_import <light.glsl>
#moj_import <fog.glsl>

in vec3 Position;
in vec4 Color;
in vec2 UV0;
in ivec2 UV1;
in ivec2 UV2;
in vec3 Normal;

uniform sampler2D Sampler1;
uniform sampler2D Sampler2;

uniform mat4 ModelViewMat;
uniform mat4 ProjMat;
uniform int FogShape;

uniform vec3 Light0_Direction;
uniform vec3 Light1_Direction;

out float vertexDistance;
out vec4 vertexColor;
out vec4 lightMapColor;
out vec4 overlayColor;
out vec2 texCoord0;

void main() {
    gl_Position = ProjMat * ModelViewMat * vec4(Position, 1.0);

    vertexDistance = fog_distance(Position, FogShape);
    vertexColor = minecraft_mix_light(Light0_Direction, Light1_Direction, Normal, Color);
    lightMapColor = texelFetch(Sampler2, UV2 / 16, 0);
    overlayColor = texelFetch(Sampler1, UV1, 0);
    texCoord0 = UV0;
}
```
{{< /details >}}

그래서 json의 vertex와 fragment 자리의 값은 다른 이름이어도 좋다. 물론 파일명을 맞춰준다는 전제 하에 말이다.

이렇게 기본 파일을 얻었다면, 이제 수정할 시간이다.

## Vertex Shader 수정
위에서 만든 vsh 파일이 바로 vertex shader이다. 이 쉐이더는 OpenGL 파이프라인의 일부로, 텍스쳐가 들어가기 전 각 꼭짓점의 위치를 정한다고 생각하면 된다.

### 파트 구별과 UV 매핑

가장 먼저 할 일은 상수 하나 둘을 추가해 주는 일이다.
```glsl
// item_display의 transformation.transition[1] 의 값을 1024*n으로 설정해 파트를 구별  
#define SPACING 1024.

// uv 위치 데이터  
const vec2[] origins = vec2[](  
vec2(40.0, 16.0), // right arm  
vec2(40.0, 32.0),  
vec2(32.0, 48.0), // left arm  
vec2(48.0, 48.0),  
vec2(16.0, 16.0), // torso  
vec2(16.0, 32.0),  
vec2(0.0, 16.0), // right leg  
vec2(0.0, 32.0),  
vec2(16.0, 48.0), // left leg  
vec2(0.0, 48.0)  
);
```
이 glsl 코드의 의미를 굳이 분석하자면, 컴파일러가 아래 코드의 `SPACING` 부분을 전부 1024. 로 바꾸게 하는 것이다. 주석에 쓰여있듯이 이 값은 파트 구별에 사용될 것이다. 굳이 1024일 필요도, y 값일 필요도 없다.

그리고 `origins`라는 vec2 배열을 생성하는데, 이는 단순히 플레이어 스킨에서 각 파트를 매핑할 때 시작할 위치를 정하게 돕는다.


```glsl
ivec2 dim = textureSize(Sampler0, 0);  
  
if (dim.x != 64 || dim.y != 64) {  
    // 플레이어 머리가 아닌 경우
    texCoord0 = UV0;  
    texCoord1 = vec2(0.0);  
    vertexDistance = fog_distance(Position, FogShape);  
    gl_Position = ProjMat * ModelViewMat * vec4(Position, 1.0);
}
```
다음 코드이다. `rendertype_entity_translucent`는 플레이어 머리뿐만 아니라 다양한 대상에 적용되므로 `Sampler0`이 가진 텍스쳐 정보를 사용해 플레이어 머리를 판별한다. 

```glsl
else {  
    vec3 wpos = Position;  
    // uv 매핑을 위한 데이터  
    vec2 UVout = UV0;  
    vec2 UVout2 = vec2(0.);  
    // head, arm_r, arm_l, torso, leg_r, leg_l 순으로 translation이 0부터 1024씩 줄어든다.  
    // 따라서 각각 0, 1, 2, 3, 4, 5 + 1/2 의 int cast가 들어온다.
    int partId = -int((wpos.y) / SPACING - 1/2);  
  
    // 파트 별로 렌더링  
    if (partId == 0) {  
        // head인 경우. 추가적 작업 불필요  
        gl_Position = ProjMat * ModelViewMat * vec4(Position, 1.0);  
        vertexDistance = fog_distance(Position, FogShape);  
    } else {
        vec4 samp1 = texture(Sampler0, vec2(54.0 / 64.0, 20.0 / 64.0));  
        vec4 samp2 = texture(Sampler0, vec2(55.0 / 64.0, 20.0 / 64.0));  
        bool slim = samp1.a == 0. || (((samp1.r + samp1.g + samp1.b) == 0.0) && ((samp2.r + samp2.g + samp2.b) == 0.0) && samp1.a == 1.0 && samp2.a == 1.);  
        // 바깥이면 1 안쪽이면 0
        int outerLayer = (gl_VertexID / 24) % 2;  
        // 한 face에서 어느 꼭짓점인지 결정  
        int vertexId = gl_VertexID % 4;
        // 한 part에서 어느 face인지 결정 : 0부터 4개씩 끊어 up down right front left back 순서  
        int faceId = (gl_VertexID / 4) % 6;  
  
        // origin을 모델 위치로 변경  
        wpos.y += SPACING * partId;  
    
        gl_Position = ProjMat * ModelViewMat * vec4(wpos, 1);  
  
  
        // uv 매핑  
        UVout = origins[2 * (partId - 1) + outerLayer];  
        UVout2 = origins[2 * (partId - 1)];  
  
        // 각 vertex별 uv 할당  
        // WARN 아래는 right arm만 지원하는 약식 구문
        vec2 offset = vec2(0.);  
        if (faceId == 0) {  
            offset += vec2(4, 4);  
            if (vertexId == 0) {  
                offset += vec2(4, 0);  
            } else if (vertexId == 1) {  
                offset += vec2(0);  
            } else if (vertexId == 2) {  
                offset += vec2(0, -4);  
            } else if (vertexId == 3) {  
                offset += vec2(4, -4);  
            }  
        } else if (faceId == 1) {  
            offset += vec2(8, 4);  
            if (vertexId == 0) {  
                offset += vec2(4, 0);  
            } else if (vertexId == 1) {  
                offset += vec2(0);  
            } else if (vertexId == 2) {  
                offset += vec2(0, -4);  
            } else if (vertexId == 3) {  
                offset += vec2(4, -4);  
            }  
        } else {  
            offset += vec2((faceId-2)*4, 4);  
            if (vertexId == 0) {  
                offset += vec2(4, 0);  
            } else if (vertexId == 1) {  
                offset += vec2(0);  
            } else if (vertexId == 2) {  
                offset += vec2(0, 12);  
            } else if (vertexId == 3) {  
                offset += vec2(4, 12);  
            }  
        }  
  
        vertexDistance = fog_distance(wpos, FogShape);  
        UVout += offset;  
        UVout2 += offset;  
        UVout /= 64.;  
        UVout2 /= 64.;  
    }
    texCoord0 = UVout;  
    texCoord1 = UVout2;  
}
```

위 if문 뒤에 붙는 else 구문인데... 참... 길다. 차근차근 살펴보자. 우선적으로 하는 일은 `wpos`와 UV 변수들을 설정하는 일이다. 그 다음 wpos.y를 통해 partId를 얻는데, 여기서 이 프로그램은 머리부터 시작해 partId가 커질수록 y값이 1024씩 **내려가는** 상황을 상정했음을 알 수 있다.

이후 머리일 경우 바닐라처럼 처리 후 넘어가지만, 다른 파트일 경우 텍스쳐와 gl_VertexID로부터 slim 스킨인 지, 바깥쪽 스킨 레이어를 설정하는 중인 지, 어떤 면을 설정하는 중인 지 등을 알아내어 일일히 매핑하고 out으로 내보내는 과정이 진행된다.

아쉽지만 이 글을 쓰는 지금 수정본 코드가 없어 slim에 따른 적용이나 다른 파트 지원이 되어있지 않다. 이는 추후 시간이 나면 교체하도록 하겠다.

### ProjMat 조작
`ProjMat`은 Projection Matrix의 줄임말로, 카메라의 입장에서 구성된 View Space를 Clip Space로 바꿔주는 역할을 한다. 이 과정에서 FOV, View Bobbing 등이 적용되므로 이 ProjMat을 적절히 조작해야 FOV에 영향을 받지 않는 애니메이션을 만들 수 있다.
![이미지](https://learnopengl.com/img/getting-started/coordinate_systems.png)

```glsl
mat4 tweakedProjMat = ProjMat;  
  
// FOV에 따른 ProjMat 변형을 조작 -> FOV 무시하고 위치 고정  
float tanFovHalf = tan(80.1 / 2.0);  
  
tweakedProjMat[0][0] /= tanFovHalf * ProjMat[1][1];  
tweakedProjMat[1][1] = tanFovHalf;  
tweakedProjMat[2][2] /= 10;  
//            newProjMat[3][0] = 0; 좌우 View Bobbing 제거  
//            newProjMat[3][1] = 0; 상하 Vie Bobbing 제거  

gl_Position = tweakedProjMat * vec4(-wpos.x, -wpos.y, wpos.z, 2);
```
위 코드에서 `wpos` 설정 이후 out `gl_Position` 값을 적용하는 부분을 이렇게 변경한다. uniform인 `Position` 대신 `wpos`를 사용한 것처럼, uniform인 `ProjMat` 대신 그 복사본을 사용한다. 그 뒤 대각선 값들을 조정하는데, 이는 ProjMat이 유도되는 과정을 알아야 한다. 필자도 설명하기 난감하니 [Reference](#Reference)를 읽도록 하자.

## 결론
사실, 필자도 몇 주 전에 적은 코드를 다시 읽으면서 잘 기억나지 않아 대충 넘긴 부분들이 존재한다. 그것들에 대해서는 필자도 더 알아보고 언젠가... 수정하러 와야겠다.

## Reference
https://github.com/bradleyq/stable_player_display/{{< br >}}
https://github.com/McTsts/Minecraft-Shaders-Wiki/{{< br >}}
https://learnopengl.com/Getting-started/Coordinate-Systems{{< br >}}
https://twlab.tistory.com/34{{< br >}}
https://zamezzz.tistory.com/66{{< br >}}


[^1]:24w05a 스냅샷 이후로 in vec3 Position 변수가 수정되고 uniform mat4 IViewRotMat이 삭제되었다. 이 둘에 대해 잘 모르겠다면 이 스냅샷 이후 버전을 사용하자.
[^2]:그런데, 왜 하필 이 파일일까? [Reference](#Reference)의 Shaders Wiki에서 Core Shader List.md 파일을 잘 살펴보자.