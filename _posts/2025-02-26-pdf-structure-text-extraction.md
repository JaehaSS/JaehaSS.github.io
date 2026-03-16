---
layout: post
title: "PDF 구조, 텍스트 추출까지"
date: 2025-02-26
description: "PDF 데이터 파싱하면서 이상한 텍스트들이 발견되어 해당 텍스트들이 왜 생겼는지 트러블 슈팅하면서 정리한 글입니다."
tags: [AI]
tistory_id: 58
---

# 개요

PDF 데이터 파싱하면서 이상한 텍스트들이 발견이 되서 해당 텍스트들이 왜 생겼는지

트러블 슈팅하면서 정리한 글입니다.

# PDF란

PDF(Portable Document Format)은 Adobe에 의해 만들었고 현재 문서 교환의 표준으로 자리 잡았습니다.

## PDF 내부 구조

### 1. Header

PDF 파일의 첫 번째 줄에 위치하고, PDF 버전 정보를 포함합니다.

> %PDF-1.3

→해당 PDF 파일의 버전이 1.3을 알려준다.

### 2. Body

Body엔 여러 객체로 구성됩니다.

객체는 트리 구조를 가지고 있고, 8가지의 기본적인 유형이 있습니다.

- 불리언(Boolean): true 또는 false
- 숫자(Number): 정수 또는 실수
- 문자열(String): (문자열) 또는 <16진수>
- 이름(Name): /이름
- 배열(Array): [요소1 요소2 ...]
- 사전(Dictionary): << /키1 값1 /키2 값2 ... >>
- 스트림(Stream): 바이너리 데이터를 포함하는 특수 객체
- 널(Null): null

### 3. Cross-reference Table(==xref Table)

PDF객체의 바이트 오프셋(파일 내 위치)을 기록하여, 파일 내에서 특정 객체를 빠르게 찾을 수 있게 합니다.

### 4. Trailer

마지막 부분에 위치하고, xref 테이블의 위치, 루트 객체의 참조 등 PDF 문서를 해석하는데 필요한 핵심 정보를 포함합니다.

# 텍스트 추출

## PDF에서 나타나는 텍스트 표현

PDF에서의 텍스트는 여러 객체 유형과 복잡한 구조로 표현됩니다.

### 1. 스트림 객체

텍스트 내용은 대부분 스트림 객체에 저장이 되고, 해당 객체는 다음과 같은 구조를 가집니다.

```
5 0 obj
<< /Length 123 >>
**stream**
BT
/F1 12 Tf
100 700 Td
(Hello World) Tj
ET
endstream
**endobj**
```

### 2. 문자열 객체

텍스트의 실제 내용은 문자열 객체로 표현이 됩니다. PDF에서 문자열은 두가지 방식으로 표현할 수 있습니다.

- 리터럴 문자열 : (Hello World)
- 16진수 문자열 <48581238F>

### 3. 폰트 객체(*)

텍스트를 렌더링하려면 폰트 객체가 필요합니다. 폰트 객체는 문자 코드를 실제 글리프(시각적 표현)에

매핑하는 정보를 포함합니다.

```
12 0 obj
<<
  /Type /Font
  /Subtype /TrueType
  /BaseFont /ABCDEF+Arial
  /FirstChar 32
  /LastChar 255
  /Widths [278 278 355 ... ]
  /FontDescriptor 13 0 R
  /Encoding /WinAnsiEncoding
	/ToUnicode Adobe-Identity-UCS
>>
```

## 폰트

### 폰트 내장

- 전체 포함 : 폰트 파일 전체를 PDF에 포함합니다
- 부분 집합: PDF문서에 사용된 글리프만 포함하여 파일 크기를 줄입니다.

### 폰트 미내장

- 표준 14 폰트: PDF뷰어에서 기본 지원하는 폰트를 사용하여 별도 폰트 파일 없이 텍스트 표시 및 추출이 가능합니다.
- 시스템 폰트 참조 : PDF가 시스템에 설치된 폰트를 참조하는 경우, 폰트 이름 일치 여부에 따라 텍스트가 깨질 수 있습니다.

### CID 폰트(CJK 폰트)

- 한글, 중국어, 일본어와 같이 글자 수가 많은 언어에서 사용됩니다.
- **CMap(Character Map)**을 사용하여 CID(Character IDentifier)를 유니코드로 변환합니다.
  - 내장 CMap : PDF에 CMap 정보가 포함된 경우
  - 외부 CMap : 드물게 CMap 정보가 PDF 외부 파일에 있는 경우
- ToUnicode CMap : CID와 유니코드 간 1:1 매핑 정보를 제공하여 가장 정확한 텍스트 추출을 가능하게 합니다.

## Python 라이브러리를 활용한 텍스트 추출

텍스트 문자열은 단순한 유니코드가 아니라, 특정 폰트의 인코딩 체계에 따른 코드 값입니다.예를 들어 한글 문서인 경우에는 CID 기반 인코딩을 사용합니다.(ex : Identity-H)

Python 라이브러리를 사용한 예시입니다.

![](https://blog.kakaocdn.net/dna/Hh3a9/btsMyIdtpi4/AAAAAAAAAAAAAAAAAAAAAPTpWFY8wPTVvWHwgTe0u6OpjtjwzZdyaemD7J_SAfYC/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=WhbC3T5eUkxFHT1ywgUl5rqImEM%3D)

```
pip install playa-pdf

with playa.open("pdf") as pdf:
    for page in pdf.pages:
        for obj in page:
            if obj.object_type == "text" and obj.chars !=" ":
                font = obj.textstate.font
                print(f"text : {obj.chars}")
                print(f"text Byte : {str(obj.args)}")
                print(f"font size : {obj.textstate.fontsize}")
                print(f"font name : {font.basefont}")
                print(f"font toUnicode : {font.tounicode.attrs['CMapName'] if font.tounicode is not None else '내장된 font 인코딩 값 사용'}")
                print("------------------------------------")

text : 유
text Byte : [b'\\x08j']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
text : 유
text Byte : [b'\\x08j']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
text : 방
text Byte : [b'\\x06\\x0f\\x00\\x03']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
text : 방
text Byte : [b'\\x06\\x0f\\x00\\x03\\x00\\x03\\x00\\x03']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
text : 생
text Byte : [b'\\x06\\xe2']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
text : 생
text Byte : [b'\\x06\\xe2']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
```

"유" 텍스트를 예를 들겠습니다.

CID의 값 : b'\\x08j'

CMap Name:/Adobe-Identity-UCS

즉 /Adobe-Identity-UCS 의 CID 값에 매칭되는 글리프의 값은 "유"라는 뜻입니다.

### 트러블 슈팅

위 예제처럼 제대로 나오는 텍스트들은 상관이 없습니다.

![](https://blog.kakaocdn.net/dna/A59Pa/btsMyA0Tioy/AAAAAAAAAAAAAAAAAAAAAJtWvbTKoQBQtxSjBaHSH-YBTg1rqBp4Embse0Y7K1TJ/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1774969199&allow_ip=&allow_referer=&signature=28effE%2BPK3SJjSGZpRFKGTt5dFk%3D)

하지만 문제가 되는 부분은 텍스트 추출을 할 떄 글자가 깨지거나 다른 값으로 추출이 됐을 떄의 케이스입니다.

```
with playa.open("pdf") as pdf:
    for page in pdf.pages:
        for obj in page:
            if obj.object_type == "text" and obj.chars !=" ":
                font = obj.textstate.font
                print(f"text : {obj.chars}")
                print(f"text Byte : {str(obj.args)}")
                print(f"font size : {obj.textstate.fontsize}")
                print(f"font name : {font.basefont}")
                print(f"font toUnicode : {font.tounicode.attrs['CMapName'] if font.tounicode is not None else '내장된 font 인코딩 값 사용'}")
                print("------------------------------------")

 text : 생
text Byte : [b'P\\xb8']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
text : 유
text Byte : [b'\\x08j']
font size : 1
font name : AAAAAF+MalgunGothic
font toUnicode : /'Adobe-Identity-UCS'
------------------------------------
```

• 문자가 → 생 이라는 글자로 변환된 케이스입니다.

해당 케이스는 [https://github.com/pdfminer/pdfminer.six/issues/1035](https://github.com/pdfminer/pdfminer.six/issues/1035) 에서 나오는 것처럼 PDF 파일이 손상됐기 떄문에 나오는 케이스입니다.

이런 현상이 발생하면 OCR 외에는 처리 방법이 없어 보입니다.

# 결론

PDF 텍스트 추출 시 다양한 문제가 발생할 수 있으며 PDF 파일 손상과 같은 경우에는 OCR과 같은 추가적인 기술이 필요할 수 있습니다.
