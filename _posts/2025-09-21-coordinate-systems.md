---
layout: post
title: "좌표계"
date: 2025-09-21
description: "개발자가 자주 만나는 좌표계(EPSG:4326, EPSG:3857, EPSG:5179 등)의 개념과 실무 적용 방법을 정리한 글입니다."
tags: [Python]
tistory_id: 65
---

## 들어가며

개발자들이 지도, 위치 기반 서비스, 게임 개발에서 다양한 좌표계 코드(EPSG, SRID)를 마주할 때 혼동을 경험합니다. 이 글은 주요 좌표계들과 그 특징, 실무 적용 방법을 다룹니다.

## 좌표계란 무엇인가?

좌표계는 "지구상의 위치를 숫자로 표현하는 방법"입니다. 구형 지구를 평면에 표현하려면 특정 규칙이 필요합니다.

### 여러 좌표계가 필요한 이유

- 지구는 완전한 구가 아닌 타원체 형태
- 거리 측정, 각도 측정, 면적 계산 등 용도별 최적화
- 지역별로 특화된 좌표계 존재

## 주요 좌표계 분류

### 지리 좌표계 (Geographic Coordinate System)

구면 좌표계로 경위도 사용:
- 위도: -90도 ~ +90도 (남극 ~ 북극)
- 경도: -180도 ~ +180도 (서경 ~ 동경)

특징: 각도 단위, 전 세계 표현 가능, 거리 계산 복잡

### 투영 좌표계 (Projected Coordinate System)

3차원 지구를 2차원 평면에 투영:
- 미터 단위 사용
- 거리/면적 계산 간단
- 특정 지역 최적화
- 투영 과정에서 왜곡

## 개발에서 자주 만나는 좌표계

### EPSG:4326 (WGS84)

- 가장 널리 사용되는 지리 좌표계
- GPS 표준
- 구글 맵, OpenStreetMap 기본값
- 전 세계 호환성 최고

```javascript
const location = {
  latitude: 37.5665,   // 서울시청 위도
  longitude: 126.9780  // 서울시청 경도
};
```

### EPSG:3857 (Web Mercator)

- 웹 지도 서비스 표준 (구글, 네이버, 카카오)
- 미터 단위
- 극지방 왜곡 심함
- 타일 기반 지도 최적화

```javascript
const webMercatorCoord = {
  x: 14135317.88,  // 서울시청 x좌표
  y: 4518386.79    // 서울시청 y좌표
};
```

### EPSG:5179 (Korea 2000)

- 한국 표준 좌표계
- 한국 지역 최적화
- 공공데이터 주요 표준
- 높은 거리/면적 계산 정확도

### EPSG:4019 (GRS 1980)

- 측지학적으로 정확한 지구 타원체
- 과학적 계산용
- 높은 정밀도 요구 시 사용

## 좌표계별 사용 사례

| 용도 | 추천 좌표계 | 이유 |
|------|----------|------|
| 전역 서비스 | EPSG:4326 | GPS 호환, 범용성 |
| 웹 지도 | EPSG:3857 | 지도 API 표준 |
| 한국 서비스 | EPSG:5179 | 국내 최적화 |
| 거리 계산 | 투영좌표계 | 미터 단위 |
| 데이터 저장 | EPSG:4326 | 호환성, 변환 용이 |

## 개발 시 주의사항

### 좌표계 변환 필수

```javascript
// 잘못된 예: 좌표계 혼용
const distance = calculateDistance(
  wgs84Coord,        // EPSG:4326
  webMercatorCoord   // EPSG:3857
);

// 올바른 예: 좌표계 통일
const wgs84Coord2 = transformCoordinate(
  webMercatorCoord,
  'EPSG:3857',
  'EPSG:4326'
);
const distance = calculateDistance(wgs84Coord1, wgs84Coord2);
```

### 라이브러리 활용 (Proj4js)

```javascript
import proj4 from 'proj4';

proj4.defs([
  ['EPSG:4326', '+proj=longlat +datum=WGS84 +no_defs'],
  ['EPSG:3857', '+proj=merc +datum=WGS84 +units=m +no_defs']
]);

const result = proj4('EPSG:4326', 'EPSG:3857', [126.9780, 37.5665]);
```

### 데이터 저장 시 좌표계 명시 (PostGIS)

```sql
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    geom GEOMETRY(POINT, 4326)
);
```

## 핵심 포인트

1. **WGS84(4326)**: 범용 표준, GPS 호환
2. **Web Mercator(3857)**: 웹 지도 표준
3. **Korea 2000(5179)**: 한국 지역 최적화
4. **좌표계 변환**: 반드시 동일한 좌표계에서 계산
5. **용도별 선택**: 서비스 특성에 맞는 좌표계 사용

## 참고 자료

- [EPSG.io](https://epsg.io/) - 좌표계 정보 검색
- [Proj4js](https://github.com/proj4js/proj4js) - JavaScript 좌표 변환 라이브러리
- [PostGIS Documentation](https://postgis.net/) - 공간 데이터베이스
