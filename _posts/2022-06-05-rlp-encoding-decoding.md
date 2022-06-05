---
title: "RLP(Recursive Length Prefix) 인코딩/디코딩"
date: 2022-06-05
classes: wide
# toc: true
# toc_sticky: true
categories:
  - blockchain
tags:
  - rlp
  - data encoding/decoding
---

## 특징
- RLP는 노드간 data 전송을 공간 효율적인 포맷으로 표준화함(그 당시엔 그러한 의도로 고안됨)
- RLP의 목적은 structure를 encoding하는 것; 특정 데이터 유형(string, float)을 인코딩하는 것은 higher-order 프로토콜에 맡겨짐\
  (= 주고 받는 입장에서, 해당 byte가 어떤 의미를 가지는지는 미리 약속되어 서로 알고 있어야한다)
- 양의 RLP 정수(=RLP 데이터로 봐도 무방)는 반드시 leading 0가 없는(=맨 앞자리에 0이 없는) big-endian binary로 표현되어야 함(이렇게 해서 정수 값 0을 empty byte array와 동일하게 만듦)
- 따라서, deserialize했을 때 leading 0들이 있는 경우는 invalid로 처리됨
- string length의 정수 표현 또한 페이로드의 정수와 마찬가지의 방식으로 인코딩 되어야함

## 정의
- RLP encoding function은 item을 받고, item은 다음과 같이 정의되어있다
	- string은 item
	- item의 리스트는 item
- 이하에서 **string은 특정 바이트 수의 binary data와 동일한 의미**를 가짐

## RLP Encoding 규칙
데이터의 종류에 따라 아래 5개의 규칙으로 나누어 encoding이 수행된다.
규칙 1~3은 단일 item에 대한 인코딩 방법이며, 규칙이 올라갈 수록 더 긴 데이터를 인코딩하는 방법이 되고, 규칙 4~5는 list에 대한 인코딩 방법이며, 이 또한 규칙5가 규칙4 보다 더 수가 많거나 더 긴 데이터를 인코딩하는 방법이 된다.
#### 규칙 1. `[0x00, 0x7f]` 범위 안의 single byte인 경우
- 해당 byte는 그 자체로 RLP 인코딩이다.
- 첫번째 byte의 범위는 `[0x00, 0x7f]`이 된다.
- e.g.) integer 15(`\x0f`)의 RLP 인코딩
	- single byte이므로, 그 자체로 표현됨
	- `[0x0f] = 0x0f`

#### 규칙 2. 단일 item이고, string(bytes)이 55 이내의 길이를 가지는 경우
- RLP 인코딩의 첫번째 byte는 `0x80` + string(bytes)의 길이이고, 그 뒤로 string의 내용이 붙는다. 
- 첫번째 byte의 범위는 `[0x80, 0xb7]`이 된다.
- 앞자리가 9, a, b이고 7 까지이니, `0x80 ~ 0xb7`은 `3*16+7=55`개까지 표현이 가능하다.
- e.g.) string "dog"의 RLP 인코딩
  - 길이가 3이므로 `0x80 + 3 = 0x83`로 시작, 뒤에 각 char 내용이 붙어 표현된다.
  - `[0x83, 'd', 'o', 'g'] = 0x83646f67`
- e.g.) integer 1024(`\x04\x00`)의 RLP 인코딩
  - byte의 길이가 2이므로 `0x80+2=0x82`로 시작, 각 byte 내용이 붙어 표현된다.
  - `[0x82, 0x04, 0x00] = 0x820400`

#### 규칙 3. 단일 item이고, string(bytes)의 길이가 55를 초과하는 경우
- RLP 인코딩의 첫번째 byte는 `0xb7` + string(bytes)의 길이를 bytes로 표현한 것의 bytes 길이, 그 뒷자리에 string의 길이를 bytes로 표현한 것, 그리고 그 뒤에 string의 내용이 들어간다. 
- 첫번째 byte의 범위는 `[0xb8, 0xbf]`이 된다.
- e.g.) char "a" 가 1024개 붙어있는 string(`"aaa...a"`)의 RLP 인코딩
  - 길이인 1024를 byte로 표현하면, `0x0400`이 되고, 이는 2바이트 이므로,  `0xb7+2=0xb9`, 이 뒤에 길이의 값에 해당하는 `0x0400`이 붙고, 이 뒤에 string의 내용이 붙음
  - `[0xb9, 0x04, 0x00, 'a', 'a', 'a', 'a', ...] = 0xb904006161616161...61616161`
- e.g.) string "Lorem ipsum dolor sit amet, consectetur adipisicing elit"의 RLP 인코딩은 글자 전체 길이가 56 이고, 56의 byte는 `0x38`이므로 1자릿 수를 가짐, 따라서 `0xb7+1=0xb8`로 시작, `0x38`이 붙고, 뒤에 순차적으로 char가 붙음 
  - `[0xb8, 0x38, 'L', 'o', 'r', 'e', 'm', ' ', ... 't'] = 0xb8384c6f72656d20697073756d20646f6c6f722073697420616d65742c20636f6e7365637465747572206164697069736963696e6720656c6974`
- `8~f` range(8개)를 통해서, 최대 `0xffffffffffffffff=18446744073709551615` 길이의 string을 담을 수 있음

#### 규칙 4. list이고, 해당 list 내 모든 item들의 RLP 인코딩 결과 길이의 합(total payload의 list)이 55개 이내인 경우
- RLP 인코딩의 첫번째 byte는  `0xc0` + list의 길이 합이 되고, 그 뒤로 각 item들의 RLP encoding이 concat되어 들어간다. 
- total payload의 list가 55개 이내라는 의미는, RLP encode한 모든 item이 55 bytes 이내와 동일한 의미이다.
- 첫번째 byte의 범위는 `[0xc0, 0xf7]`이 된다.
- e.g.) `["cat", "dog"]`의 RLP 인코딩 
  - `["cat", "dog"]`의 각 item들의 RLP encoding은 `0x83, 'c','a','t', '0x83', 'd','o','g'`이고,
  -  total payload의 list의 길이가 8이므로, `0xc0+8=0xc8`이 첫번째 byte의 값이 되고, 그 뒤로 리스트의 결과가 붙는다.
  - `[0xc8, 0x83, 'c', 'a', 't', 0x83, 'd', 'o', 'g'] = 0xc88363617483646f67`

#### 규칙 5. list이고, 해당 list 내 모든 item들의 RLP 인코딩 결과 길이의 합(total payload의 list)이 55개 초과인 경우 
- RLP 인코딩의 첫번째 바이트는 `0xf7` + payload 길이를 바이트 형태로 표현한 것의 길이가 되고, 그 뒤로 payload의 길이의 바이트 값이 붙고, 그 뒤에 item의 RLP encoding 값이 붙는다.(규칙 3와 유사)
- 첫번째 byte의 범위는 `[0xf8, 0xff]` 이다.
- e.g.) a가 연속으로 50개, b가 연속으로 50개인 두개의 element를 가지는 리스트 `["aaa..a", "bbb...b"]` 인 RLP 인코딩 
  - 첫번째 item에서 `'a'`는 `\x61`, 50개는 0x80 + 0x32 뒤에 61 50개, 0xb261616161... ; byte로 51자리
  - `'b'`는 `62, 50개 또한 0x80 + 0x32 뒤에 62 50개, 0xb262626262..50개; byte로 51자리
  - 아이템의 rlp 인코딩은 51 * 2, 102자리, byte 표현은 0x66으로 1byte를 차지함
  - `0xf7+1=0xf8`로 시작, 뒤에 실제 길이의 byte 표현인 `0x66`, 뒤에 string(a50개의 byte표현(`b261616161...`), b50개의 byte표현(`b262626262...`))이 붙는다
  - `[0xf8, 0x66, 0xb2,  'a', 'a', ... , 'a', 0xb2, 'b', 'b', ..., 'b'] = 0xf866b2616161...61b2626262...62`


## RLP Decoding 과정
1. 입력 데이터의 첫번째 바이트(prefix) 종류에 따라서, data type, 실제 데이터의 길이와 offset을 decode 한다
2. 데이터의 type과 offset으로 data를 decode 한다
3. 나머지 입력을 decode한다

데이터의 type과 offset은 첫번째 byte의 값에 따라 아래의 규칙으로 분류할 수 있다. (이는 encoding 규칙과 동일하다)

#### 규칙 1. 첫번째 byte가 `[0x00, 0x7f]`이면, 단일 item이며, byte 값 그 자체가 데이터가 된다
- string은 첫번째 byte 값 그 자체임

#### 규칙 2. `[0x80, 0xb7]`이면, 단일 item이며, 55자리 이하의 길이의 데이터로 구성되어 있다
- string의 길이는 `첫번째 바이트 - 0x80`이 되고, 해당 길이만큼의 string을 읽으면 됨

#### 규칙 3. `[0xb8, 0xbf]`로 첫번째 바이트가 시작하면, 단일 item이며 55자리 초과의 길이의 데이터로 구성되어 있다
- string의 바이트 표현의 길이는 `첫번째 바이트 - 0xb8`이고, 해당 길이만큼의 뒤에 따라오는 byte를 읽어서 실제 string의 길이를 decode한 후에, 해당 길이만큼의 string을 읽으면 됨

#### 규칙 4. `[0xc0, 0xf7]`로 첫번째 바이트가 시작하면, list이며, 리스트 내 모든 item들의 RLP 인코딩 결과 길이의 합이 55자리 이하의 데이터로 구성되어 있다.
- 리스트 내 모든 item들의 RLP encoding 결과의 길이의 합이 `첫번째 바이트 - 0xc0`이 되고, 해당 길이만큼의 string을 읽으면 됨(리스트가 nest된 경우에도 동일한 기준으로 적용)

#### 규칙 5. `[0xf8, 0xff]`로 첫번째 바이트가 시작하면, list이며, list 내 모든 item들의 RLP 인코딩 결과 길이의 합이 55자리 초과의 데이터로 구성되어 있다.
- 리스트 내 아이템의 갯수를 표현하는 바이트의 길이가 `첫번째 바이트 - 0xf7`이고,  해당 길이만큼의 뒤에 따라오는 byte를 읽어서 실제 리스트 내 아이템의 갯수를 decode한 후에, 해당 이만큼 string을 읽으면 됨

## Reference
[Ethereum wiki](https://eth.wiki/fundamentals/rlp)