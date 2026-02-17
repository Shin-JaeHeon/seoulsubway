# seoulsubway
Interactive Isochrone Map of Seoul Subway 
대한민국 수도권 지하철 반응형 시간거리지도

다음의 주소에서 반응형 웹페이지를 볼 수 있습니다.
https://vuski.github.io/seoulsubway/

다음의 주소에 관련 내용이 설명되어 있습니다.
https://www.vw-lab.com/68

## 실제 한국 지도(네이버/구글/위성) 덧붙이기 가이드

이 프로젝트의 역 좌표는 **EPSG:5179 (UTM-K, Korea 2000 Unified CS)** 기반이다. 따라서 네이버/구글처럼 일반적으로 WGS84(`lat/lng`)를 쓰는 지도 SDK를 붙일 때는 좌표변환이 핵심이다.

### 1) 권장 아키텍처
- 지도 SDK(div)를 **배경 레이어**로 둔다.
- 기존 `glcanvas`/`text` 캔버스는 **오버레이 레이어**로 유지한다.
- 최단거리/등시선 계산(WebGL)은 그대로 두고, 뷰포트(중심/배율)만 지도와 동기화한다.

### 2) 레이어 구성 예시
```html
<div class="container">
  <div id="map"></div>
  <canvas id="glcanvas"></canvas>
  <canvas id="text"></canvas>
</div>
```

```css
.container { position: relative; width: 100vw; height: 100vh; }
#map { position: absolute; inset: 0; z-index: 0; }
#glcanvas { position: absolute; inset: 0; z-index: 10; }
#text { position: absolute; inset: 0; z-index: 20; pointer-events: auto; }
```

### 3) 좌표계 변환 (필수)
`proj4`를 사용해 5179↔4326 변환 함수를 만든다.

```js
const EPSG5179 = '+proj=tmerc +lat_0=38 +lon_0=127.5 +k=0.9996 +x_0=1000000 +y_0=2000000 +ellps=GRS80 +units=m +no_defs';

function tmToLngLat(x, y) {
  const [lng, lat] = proj4(EPSG5179, 'EPSG:4326', [x, y]);
  return { lat, lng };
}

function lngLatToTm(lng, lat) {
  const [x, y] = proj4('EPSG:4326', EPSG5179, [lng, lat]);
  return { x, y };
}
```

### 4) 동기화 전략
- 현재 코드의 카메라 상태는 `cenX`, `cenY`, `maxX-minX`(사실상 축척)로 관리된다.
- `cenX/cenY`가 바뀔 때마다 지도의 `center`를 갱신한다.
- 지도 줌/이동 이벤트가 발생하면 역으로 `cenX/cenY`와 축척을 갱신하고 `render()`를 호출한다.
- 기존 마우스 휠/드래그를 유지하려면, 이벤트 소유권을 `text` 캔버스에 두고 지도는 배경 렌더러로만 쓰는 방식이 가장 단순하다.

### 5) 지도 선택지
- **네이버 지도 SDK**: 국내 상세도와 한국 사용자 친화성이 좋다.
- **구글 지도 SDK**: 문서/예제가 많고 글로벌 확장에 유리하다.
- **위성 지도**: 네이버/구글의 satellite 타입 또는 VWorld 위성 타일 사용 가능.

### 6) 시각화 튜닝 포인트
- 등시선 면 알파를 조금 더 낮추면(채도/명도도 완화) 배경 지형과 노선 가독성이 함께 좋아진다.
- 노선색 인지가 핵심이므로 역/선 대비를 유지하고, 등시선은 보조 정보로 두는 편이 안정적이다.

### 7) 라이선스/약관 주의
- 지도 타일을 직접 크롤링해 WebGL 텍스처로 붙이는 방식은 약관 이슈가 생기기 쉽다.
- 네이버/구글/VWorld는 **공식 SDK/공식 타일 사용 정책**에 맞춰 구현해야 한다.

### 8) 데이터 관점 메모
- 본 프로젝트 설명처럼 철도 네트워크는 환승 노드를 분리하고, 방향성/대기시간/운행빈도 차이를 반영할수록 결과 해석력이 높아진다.
- 등시선(Delaunay 기반)은 역에서 멀리 떨어진 빈 공간이 과장될 수 있으므로, 필요 시 dummy 포인트를 넣어 보정한다.
