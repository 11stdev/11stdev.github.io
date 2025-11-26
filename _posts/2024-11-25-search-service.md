---
layout: post
title: 검색 서비스에서 좋은 품질의 코드를 찾는 은하수 항해 기록
author: 검색FE
date: 2024-11-25
tags: [frontend, React]
---

# 들어가며

<br/>
안녕하세요.
11번가 검색/추천 서비스 개발팀에서 11번가 검색 서비스의 프런트엔드 개발을 담당하고 있는 **김다미**, **이호찬**입니다.
검색 서비스의 프런트엔드 파트에서는 검색 결과를 큐레이션하여 사용자가 원하는 상품을 더욱 쉽게 탐색할 수 있도록 돕는 다양한 형태의 UI를 개발하고 있습니다.

11번가 검색 서비스는 사용자의 클릭과 구매 전환율 등 다양한 활동 지표를 바탕으로, 더 나은 탐색 경험을 제공하기 위해 지속적으로 발전하고 있습니다.
또한, 상품을 탐색하는 사용자와 상품을 연결하는 핵심 통로로서, 전시, 상품상세, 주문, 배송, 혜택, 아마존 등 다양한 도메인의 신규 정책에 발맞춰 빠르게 변화해야 하는 역할을 담당하고 있습니다.

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/1-search-ui-type.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/1-search-ui-type.png">
    </a>
    <figcaption class="caption">검색 결과 페이지</figcaption>
</figure>

이러한 변화에 **최소한의 비용으로 빠르게 대응하기 위해, 구조를 ‘민첩성’과 ‘유연성’을 갖춘 형태로 개선**하고자 했습니다.
그러나 기존 검색 서비스의 PC와 모바일 코드는 상반된 설계 스타일을 가지고 있었기에, 어느 스타일을 기준으로 통합할지에 대한 깊은 고민이 필요했습니다.

이 과정에서 **'어떤 코드가 좋은 품질의 코드인가?'라는 질문을 끊임없이 던지며, 검색 서비스의 특성에 적합한 컴포넌트를 설계하고 개선해 온 경험을 이번 글에서 공유하고자 합니다.**

# 검색 리스팅과 컬렉션

검색 페이지에서 상품은 크게 **(1) 리스팅**과 **(2) 컬렉션**이라는 두 가지 유형으로 구분됩니다.

**리스팅은 검색 페이지의 표준 UI로, 여러 상품을 묶어 나열하는 방식**입니다.
상단에는 상품의 특성을 플래그나 텍스트 형태로 표시하고, 중간에는 주요 상품 정보를, 하단 확장 영역에는 리뷰와 같은 부가 정보를 노출하는 구조로 설계되어 있습니다.

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/2-listing.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/2-listing.png">
    </a>
</figure>

**컬렉션은 특정 목적에 따라 상품을 큐레이션하여 다양한 형태로 보여주는 UI**로, 각각의 컬렉션은 고유한 개성을 갖추고 있습니다.

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/3-search-collections.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/3-search-collections.png">
    </a>
</figure>

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/4-search-collections.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/4-search-collections.png">
    </a>
</figure>

<figure style="width: 40%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/search-never-stop-2.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/search-never-stop-2.png">
    </a>
</figure>

이 외에도 아마존타임딜컬렉션 아마존핫딜컬렉션 오픈딜컬렉션 쇼킹딜컬렉션 장보기컬렉션 구매가이드배너 팁콕 기획전 베스트리뷰 30일내최저가컬렉션 큐레이션컬렉션
브랜드동영상광고 브랜드이미지광고 JBP배너 정답형컬렉션 패키지형컬렉션 반복구매컬렉션 아마존반복구매컬렉션 예약/공동구매컬렉션 방송상품컬렉션
브랜드검색DA광고이미지형 브랜드검색DA광고스토어형 포커스클릭 연관상품 연관키워드 애플컬렉션 T공식대리점컬렉션 십일초이스컬렉션 타임딜컬렉션
예약/공동구매컬렉션 가격비교컬렉션 보이스콘텐츠컬렉션 딜컬렉션 라이브11컬렉션 모델명컬렉션 팁콕/콘텐츠컬렉션 관심테마컬렉션 다른고객이함께본상품컬렉션
브랜드DA컬렉션 항공권/렌터카컬렉션 아마존십일절컬렉션 오늘의딜컬렉션 베스트꾹꾹컬렉션 백화점컬렉션 아웃렛컬렉션 홈쇼핑컬렉션...

사용자의 탐색 경험을 개선하기 위해 많은 노력과 끊임없는 시도가 이어졌습니다. 좋은 코드에 대해 고민을 시작할 무렵, 검색 프런트 코드에는 무려 **63개의 컬렉션이 존재**했습니다.

코드는 작성 당시의 요구사항과 고유의 설계 철학을 반영하고 있습니다. 따라서 **개선 전 작성 당시의 코드에 담긴 의도를 이해하는 것이 중요**합니다.
이러한 이해를 바탕으로, 기존 코드의 장점을 유지하면서도 현재의 요구사항에 맞는 개선 방향을 모색하고자 했습니다.
특히, **코드가 해결하려던 문제와 당시의 상황적 한계, 기술적 한계를 파악하여 불필요한 변경을 최소화**하고자 했습니다.

# 검색 페이지의 변천사

컬렉션은 기존 상품 정보와 차별화된 독창적인 UI로 사용자의 시선을 효과적으로 사로잡습니다.
그러나 컬렉션의 개수가 점차 증가하면서 노출 방식과 영역 크기가 서로 달라 혼잡한 느낌을 주기 시작했습니다.
또한 좌우 스와이프나 전체보기 랜딩과 같은 다양한 액션들은 사용자 피로도를 높여 탐색 흐름을 방해하는 문제로 이어졌습니다.

이 문제를 개선하기 위해 2023년에는 리스팅과 컬렉션 간의 공통 가이드를 수립하여 통합 검색 내 UI의 일관성을 높이는 UI 개선 프로젝트를 진행했습니다.
이에 맞춰 검색 프런트엔드 영역에서도 컴포넌트 설계를 재정비하는 시간을 가졌습니다.

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/3-search-collections.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/2024-11-26-search-collections2.png">
    </a>
</figure>

우선 변경이 자주 발생했던 영역과 그렇지 않은 영역을 살펴보았습니다. 각 컬렉션은 독립적으로 변경될 때도 있었으며, 다음과 같은 기획 요청이 개별적으로 추가되는 경우가 많았습니다.

| 요청사항                                     | 적용 대상    | 요약           |
| -------------------------------------------- | ------------ | -------------- |
| 딜 컬렉션에서는 할인율 미노출                | 딜 컬렉션    | 조건부 미노출  |
| 모든 컬렉션 내 배송정보를 가격 옆으로 옮기기 | 모든 컬렉션  | 공통 구조 변경 |
| 특정 검색 탭에서는 상품 옵션 미노출          | 특정 검색 탭 | 탭별 조건 처리 |

**변경 요청은 공통된 규칙이 적용되는 경우도 있었지만, 특정 컬렉션이나 탭에만 국한된 경우도 있었습니다.**
이를 바탕으로 변경 단위를 세분화하고, 각 단위에 맞는 컴포넌트로 추상화하는 방향으로 설계를 진행했습니다.

**적절한 추상화**는 코드의 가독성과 유지보수성을 높이며, 오류에 대한 견고함을 강화해줍니다.
나이가 핵심적인 부분에 집중할 수 있게 되어, 기술적 복잡도를 효과적으로 낮출 수 있습니다.

## 기존 코드 의도 파악하기

검색 서비스 주요 코드인 React로 예시를 들어서 설명하겠습니다.

다음은 검색 서비스 페이지 **(1) PC 코드**와 **(2) 모바일 코드**의 첫인상입니다.
우선 구조적 차이에 주목해봅시다.

```jsx
// (1) PC 코드
const item = () => {
  return (
    //...
    <Price
      finalPrc={finalPrc}
      unitPrc={unitPrc}
      unitPrcInfo={unitPrcInfo}
      unitTxt={unitTxt}
      optPrcText={optPrcText}
      discountText={discountText}
      pricePrefix={pricePrefix}
      selPrc={selPrc}
      lowest
    />
    //...
  );
};
```

```jsx
// (2) 모바일 코드
const item = () => {
  return (
    //...
    <div className="c-card-item__price-info">
      <dt>가격정보</dt>
      <dd className="c-card-item__rate">
        <span className="c-card-item__special-bargain">{item.discountText}</span>
      </dd>
      <dd className="c-card-item__price-del">
        <span className="sr-only">판매가</span>
        <del>{item.selPrc}원</del>
      </dd>
      <dd className="c-card-item__price-lowest">
        <em>최저</em>
        <strong>{item.finalPrc}</strong>
        {item.unitTxt}
        {item.optPrcText}
      </dd>
    </div>
    //...
  );
};
```

**과거 PC 코드**는 각 공통 영역을 추상화하여 구현되었으며, **공통된 영역이 있다면 모든 곳에서 동일한 공통 컴포넌트를 재사용하도록 설계**되었습니다.

반면, **과거 모바일 코드**는 공통 영역을 추상화하지 않고 **컬렉션별로 독립적으로 설계**되었습니다.
이러한 구조는 공통적으로 변경이 필요될 때 수정해야 할 파일이 많아지지만, 하나의 변경에 대해서는 수정이 비교적 쉽습니다.

**검색 서비스 도메인은 리스팅과 각각의 컬렉션이 별도의 정책에 따라 변경되기 때문에, 단순히 중복 코드를 제거해 공통화를 시도할 경우 모듈 간 의존성이 강해져 유지보수가 어려워질 수 있습니다.**
반면, 적절한 공통화 없이 코드를 작성하면, 추후 일괄적인 변경 시 관리 포인트가 늘어나고 영향도를 파악하기 어려워 놓치는 부분이 발생할 가능성이 높아집니다.

# 컴포넌트 재설계: 명확한 책임 분리와 구조 개선

문제를 해결하기 위한 첫 단계로, 리스팅의 명확한 책임 분리와 구조 개선 작업을 시작했습니다.

검색 서비스의 리스팅은 상품 정보를 보여주는 주요 UI 컴포넌트로, API에서 제공하는 다양한 상품 정보를 조합해 노출하는 구조를 가지고 있습니다. 이를 단순화하면 다음과 같은 단위로 나눌 수 있습니다.

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/2024-11-26-listing-3.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/2024-11-26-listing-3.png">
    </a>
</figure>

모든 검색 탭의 리스팅은 공통된 상품 정보를 조합해 노출하지만, 추상화 수준이 일정하지 않거나 공통화되지 않은 영역이 많았습니다. 각 탭이 개별적으로 구현되면서 동일한 의도를 가진 코드가 점차 다른 방식으로 작성되었고, 동일한 정책을 반영할 때도 모든 탭에 반복적인 수정이 필요했습니다.

이 문제를 해결하기 위해 변경되는 **정책의 최소 단위인 '상품명', '가격 정보', '배송 정보'로 코드를 추상화하여 컴포넌트를 생성**하고, 각 영역에 명확한 책임을 부여했습니다.

## 적절한 props 설계하기

Props 설계는 컴포넌트의 **결합도**와 **재사용성**을 좌우하는 중요한 요소이기에, 개발 과정에서 **어떤 범위까지 Props를 전달할지**는 자주 직면하는 문제입니다.

검색 API에서 제공되는 item 객체는 상품 노출에 필요한 다양한 데이터를 포함하고 있습니다.
이 데이터를 검색 리스팅을 구성하는 컴포넌트에 전달할 때, **(1) item 전체를 Props로 전달해 내부에서 필요한 값을 추출하는 방식** 과 **(2) 필요한 값만 명시적으로 전달받는 방식** 중 어느 설계가 검색 서비스 변경사항에 더 유연하게 대응할 수 있을지 고민했습니다.

```jsx
// item 전체를 전달하는 방식
<Delivery item={item} />

// 명시적으로 전달하는 방식
<Delivery deliveryInfo={item.deliveryInfo} estimatedTimeOfArrival={item.estimatedTimeOfArrival} />
```

### item 전체를 전달하는 방식

item 객체 전체를 전달하면 데이터 필드명이 변경되거나 새로운 필드가 추가될 경우에도 컴포넌트 내부에서 필요한 값을 선택적으로 추출할 수 있어 편리합니다.

```jsx
const Delivery = ({ item }) => {
  const { deliveryInfo, estimatedTimeOfArrival, sendTodayInfo } = item;
  //...
};
```

하지만 필요한 데이터를 외부에서 명확히 알 수 없기 때문에 컴포넌트 의존성 파악이 어려워지고, 테스트 시 데이터를 모킹(mocking)할 때도 불필요한 필드까지 포함해야하는 문제가 있었습니다.

### 명시적으로 전달하는 방식

명시적으로 Props를 전달하면 컴포넌트의 의존성이 명확해지고, 독립적인 관리와 테스트가 용이합니다.
사용하는 데이터만 Props로 정의하기 때문에, 컴포넌트를 사용하는 개발자가 어떤 데이터가 필요한지 한눈에 파악할 수 있습니다.
그러나 상위 컴포넌트에서 전달해야 할 필드가 많아질수록 관리 부담이 커지고, Props의 길이가 길어지며 가독성이 떨어질 가능성도 있습니다.

```jsx
<AmazonPrice
  is11stLowPrcPrd={is11stLowPrcPrd}
  is30DayLowPrcPrd={is30DayLowPrcPrd}
  is30DayCheaperMidPrcPrd={is30DayCheaperMidPrcPrd}
  isDisplayDscPrc={isDisplayDscPrc}
  finalPrc={finalPrc}
  selPrc={selPrc}
  finalDscRt={finalDscRt}
  prcDscRt={prcDscRt}
  strFinalDscRt={strFinalDscRt}
  prdNo={prdNo}
  midPrcPrd={midPrcPrd}
  discountPrice={discountPrice}
  areaName={areaName}
  isDealProduct={isDealProduct}
  logData={logData}
  //...
/>
```

| 방식               | 장점                                           | 단점                                     |
| ------------------ | ---------------------------------------------- | ---------------------------------------- |
| **item 전체 전달** | 데이터 구조 변경에 유연, 내부에서 값 추출 가능 | 의존성 파악 어려움, 테스트 Mocking 복잡  |
| **명시적 전달**    | 의존성 명확, 관리와 테스트 용이                | Props 길이 증가, 상위 컴포넌트 관리 부담 |

### 결론

많은 팀내 고민 결과,
**(1) 정책의 최소 단위**인 '상품명', '가격 정보', '배송 정보' 등의 **컴포넌트에 넘겨야할 props가 4개 이상인 경우에만 item 전체를 props로 전달**받고,
**(2) 나머지 경우의 컴포넌트와 정책의 최소 단위를 구성하는 하위 컴포넌트에서는 사용하는 값만 props로 전달받아 처리하는 방식**으로 설계하기로 결정했습니다.
선택한 방식이 변경 사항에 대한 유연성과 컴포넌트 의존성의 명확성을 동시에 확보할 수 있어, 장기적인 유지보수와 확장성을 높이는 데 적합하다고 판단했습니다.

```jsx
// 상품 정보를 구성하는 컴포넌트
<CardItemInfo>
  <FlagList infoText={item.infoText} brandBi={item.brandBi} isAdItem={isAdItem} />
  <ItemBrand isOfficial={item.isOfficial} brandEngNm={item.brandEngNm} />
  <ItemName title={item.title} listingPrdTypeText={item.listingPrdTypeText} />
  <ItemPrice item={item} />
  //...
</CardItemInfo>
```

상위 컴포넌트에서는 item을 넘겨 '유연성'을 확보하여, API 구조 변경이나 정책 갱신에 최소한의 비용으로 빠르게 대응할 수 있었습니다.

```jsx
// 배송 컴포넌트를 구성하는 하위 컴포넌트 중 하나
<ItemDeliveryInfo deliveryInfo={item.deliveryInfo} estimatedTimeOfArrival={item.estimatedTimeOfArrival} />
```

상위 컴포넌트를 구성하는 하위 컴포넌트에서는 최대한 낮은 결합도를 유지하기 위해 props 외에 다른 의존성을 갖지 않게 설계했습니다.
또한, 너무 많은 props가 존재하지 않도록 하나의 책임만 최대한 가지도록 개선했습니다.

## 상품 정보 노출 책임 분리하기

검색 서비스 페이지에서 상품 정보의 노출 여부는 **(1) API에서 제공하는 값**과 **(2) 리스팅 및 컬렉션 타입별 조건**에 따라 달라집니다. 이때, 노출 조건을 어디서 처리할지에 대한 설계는 코드의 가독성과 유지보수성에 큰 영향을 미칩니다.

### 상품 정보 노출 책임: 부모 컴포넌트 vs 자식 컴포넌트

#### 부모 컴포넌트에서 조건 처리

조건부 렌더링의 경우, 부모 컴포넌트에서 조건을 처리하는 것이 더 직관적이고 편리합니다.
부모 컴포넌트에서 어떤 UI가 렌더링되는지 한눈에 파악할 수 있으며, 여러 자식 컴포넌트를 조율하기에도 적합합니다.

```jsx
// 부모 컴포넌트에서 조건부 렌더링
const PriceExample = () => {
  return (
    <CardItemLowest>
      <CardItemLowestText text={priceText} />
      {(priceText || /* 버튼 노출을 위한 여러 조건의 조합..*/) && (
        <HelpButton
          layerType={layerType}
          infoAdditionalContents={getInfoAdditionalContents()}
          buttonText="가격안내 도움말"
          logData={logData}
        />
      )}
    </CardItemLowest>
  );
};
```

- **장점**
  - 조건이 직관적이며 렌더링 흐름을 쉽게 파악할 수 있음.
- **단점**
  - 조건이 복잡해질수록 부모 컴포넌트가 비대해지고 가독성이 저하됨.
  - 조건이 변경될 때마다 부모 컴포넌트를 수정해야 하므로 컴포넌트 간 의존성이 증가함.

### 자식 컴포넌트에서 조건 처리

컴포넌트 내부에서 사용하는 값들로 노출 유무가 결정이 된다면, 자식 컴포넌트 내부에서 노출 여부를 판단하도록 설계하면 부모 컴포넌트의 복잡도를 줄이고, 유지보수를 더 쉽게 할 수 있습니다.

```jsx
const ProductBenefit = ({ saleDescriptions }) => {
  if (!saleDescriptions) return null;

  return (
    <div className="product-benefit">
      <p>{saleDescriptions}</p>
    </div>
  );
};
```

- **장점**
  - 부모 컴포넌트의 코드가 간결해지고 가독성이 향상됨.
  - 조건이 변경되더라도 자식 컴포넌트만 수정하면 되므로 유지보수가 수월함.
  - 책임이 명확히 분리되어 컴포넌트의 재사용성이 높아짐.
- **단점**
  - 자식 컴포넌트의 내부 구현을 확인해야 노출 조건을 파악할 수 있음.
  - 부모가 모든 조건을 관리하지 않으므로 흐름이 분산될 가능성.

## 상황에 맞는 설계 선택

부모 컴포넌트와 자식 컴포넌트 간의 조건 처리 방식은 조건의 복잡도와 관리 주체에 따라 설계해야 합니다. 단순한 조건이나 여러 자식 컴포넌트를 통합적으로 관리해야 하는 경우에는 부모 컴포넌트에서 처리하고, 독립적이고 데이터 의존성이 명확한 조건인 경우 자식 컴포넌트에서 처리하는 것이 좋습니다.

### 부모 컴포넌트에서 처리하는 경우

검색 서비스에 API 데이터와 디자인이 동일한 가격 정보 영역은 공통 컴포넌트로 통합될 수 있으나, 컬렉션의 종류와 상품이 노출되는 탭에 따라 가격 정보의 노출 방식이 달라집니다.

<figure style="width: 90%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/price-type2.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/price-type2.png">
    </a>
</figure>

가격 정보를 구성하는 자식 컴포넌트들이 각각 노출 조건에 필요한 값을 Props로 전달받아야 한다면 어떻게 될까요?

조건이 추가되고 복잡성이 증가할수록 자식 컴포넌트는 점점 더 많은 Props를 요구하게 되며, 이는 자식 컴포넌트가 부모 컴포넌트와 높은 결합도를 가지게 되는 결과로 이어집니다. 따라서 이 경우에는 가격 정보를 관리하는 부모 컴포넌트에서 복잡한 노출 조건을 처리해주는 것이 좋습니다.

특히, Props 변수명에 is가 prefix로 붙는 경우, 이러한 높은 결합도의 경향은 더욱 두드러졌습니다. is로 시작하는 논리 값들이 많아질수록, 자식 컴포넌트가 부모 컴포넌트의 상태나 로직에 과도하게 의존하게 되는 상황이 자주 발생했기 때문입니다.

### 자식 컴포넌트에서 처리하는 경우

이 외 조건이 특정 자식 컴포넌트의 동작에만 관여하고, 다른 자식 컴포넌트와 독립적으로 처리될 수 있는 경우에는 자식 컴포넌트 내부에서 노출 여부를 판단하도록 설계하여 자식 컴포넌트가 자신의 책임을 명확히 가지는 방향이 적절합니다.

<figure style="width: 90%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/price-component.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/price-component.png">
    </a>
</figure>

## 상품 정보 노출을 제어하는 CollectionVisibilityManager

11번가의 상품은 추천 상품, 일반 상품, 광고 상품 등 다양한 종류로 구성됩니다.
백엔드에서는 각 상품 종류에 따라 서로 다른 엔진을 호출해 상품 정보를 생성하고, 이를 프런트엔드에 제공합니다.

그러나 상품 종류에 따라 API에서 제공되는 정보가 다를 수 있으며, 동일한 정보라도 기획 사항 변경에 따라 노출 여부가 달라지는 상황이 발생하기도 합니다.

또한 기존 코드에는 더 이상 사용되지 않는 상품 정보 관련 코드가 남아 있어 유지보수를 어렵게 만들었습니다.
특히 검색 서비스는 키워드에 따라 노출되는 상품 정보가 동적으로 변하기 때문에, 이러한 불필요한 코드를 찾아 제거하는 작업이 더욱 복잡하고 번거로웠습니다.

이 문제를 해결하고 리스팅의 상품 종류와 탭별 노출 여부를 일관되게 관리하기 위해,
**CollectionVisibilityManager**를 두어 노출 여부를 제어하도록 했습니다.

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/2024-11-26-collectionManager.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/2024-11-26-collectionManager.png">
    </a>
</figure>

```ts
class CollectionVisibilityManager {
  private readonly collectionType: ListType;
  private readonly tabId: TabId;

  static readonly visibleItemOptionTab = [
    /*...*/
  ];
  static readonly visibleItemOptionCollection = [
    /*...*/
  ];

  constructor(collectionType: ListType, tabId, TabId) {
    this.collectionType = collectionType;
    this.tabId = tabId;
  }

  isVisibleItemOption(): boolean {
    if (!CollectionVisibilityManager.visibleItemOptionTab.includes(tabId)) return false;
    return CollectionVisibilityManager.visibleItemOptionCollection.includes(this.collectionType);
  }
  //상품 옵션 노출 여부를 판단하는 메서드
  //...
}
```

`CollectionVisibilityManager`는 상품 종류와 탭에 따른 노출 여부를 명확하게 정의하고 관리하는 책임을 갖습니다.

```jsx
const collectionVisibility = new CollectionVisibilityManager(itemType, tabId);

collectionVisibility.isVisibleItemOption() && <ItemOption />;
```

새로운 상품 종류나 탭이 추가될 경우 CollectionVisibilityManager에 규칙만 추가하면 되기 때문에 복잡도가 낮아지고 관련 로직 수정도 최소화됩니다.
동시에 명시적으로 관리되기 때문에 유지보수 부담이 줄어듭니다.

# 컬렉션의 확장 가능성 확보하기

## 중복과 분리의 균형: 컴포넌트 재사용과 새로 만들기의 기준

리스팅과 완전히 동일한 UI를 가지지 않는 컬렉션은 어떤 구조로 설계하는것이 좋을까요?

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/3-search-collections.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/2024-11-26-search-collection-listing-compare.png">
    </a>
</figure>

컬렉션은 큐레이션 목적에 따라 상품 정보의 노출 여부, 형태, 순서를 조정하여 적합한 정보를 제공합니다.
리스팅에서 만들어 놓은 컴포넌트를 조합해 컬렉션을 구성할 수 있습니다.

컬렉션은 기존 컴포넌트를 활용하는 것이 좋을까요? 아니면 새로 만드는 것이 좋을까요?

리스팅의 가격 정보 컴포넌트인 `ItemPrice`의 내부 구성을 간소화하여 다음과 같다고 해봅시다.

```react
const ItemPrice = () => {
    return (
        <CardItemPriceInfo>
            <CardItemPriceSpecial />
            <CardItemRate />
            <CardItemPrice />
            <CardItemPriceDelivery />
            <CardItemPriceBenefit />
        </CardItemPriceInfo>
    );
}
```

컬렉션은 독립적인 단위로 정책이 결정됩니다. 즉, 리스팅과 동일한 구조로 상품 정보를 보여주지 않을 수 있습니다.
리스팅과 완전하게 동일한 구조를 갖지 않는다면 중복된 코드가 존재하게 되더라도 각 컬렉션 안에서 새로 컴포넌트를 만드는 것이 추후 변경사항에 유리합니다.

예를들어, 추천 컬렉션에서 CardItemPrice만 노출하도록 정책이 결정 되었다고 합시다.
`ItemPrice` 컴포넌트 내부에서 조건을 분기하여 추가하는 것보다 추천 상품 가격 전용 컴포넌트(`ItemRecommendPrice`)를 새로 만드는 것이 유지보수 측면에서 좋습니다.

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/3-search-collections.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/2024-11-26-search-itemPrice.png">
    </a>
</figure>

```react
const ItemRecommendPrice = () => {
    return (
        <CardItemPriceInfo>
            <CardItemPrice />
        </CardItemPriceInfo>
    );
}
```

이때, `CardItemPriceSpecial`, `CardItemRate`, `CardItemPrice`, `CardItemPriceDelivery`, `CardItemPriceBenefit`는 가격 영역을 구성하는 최소 단위의 요소들이므로 컬렉션 별로 변경될 가능성이 적습니다.
재사용해 줍시다.
추후에 추천 컬렉션에 `CardItemPriceSpecial`가 추가된다면, 그때 `CardItemPriceSpecial`를 추가하고, 리스팅과 노출 조건이 동일해진다면 그제서야 `ItemPrice` 컴포넌트를 사용하는것이 좋습니다.

## Wrapper 컴포넌트로 마크업 공통화하기

11번가에서는 전사 공통 마크업을 관리하는 조직이 있으며, 검색 서비스 개발팀도 이러한 공통 마크업 구조를 기반으로 컴포넌트를 개발합니다.
마크업 정책에 따라 태그 구조나 공통 클래스명이 변경되거나, 신규 UI가 기존 마크업 구조를 재사용하는 경우가 빈번했습니다.
이러한 변화에 유연하게 대응하고, 유지보수성을 높이기 위해 **Wrapper 컴포넌트**를 도입하여 마크업 구조를 공통화하고 재사용성을 강화했습니다.

Wrapper 컴포넌트는 `Thumbnail`, `HeadLine`, `GridRow`, `GridColumn` 등 **레이아웃 개발에서 반복적으로 사용되는 마크업 구조를 추상화한 컴포넌트**입니다.
이 컴포넌트들은 클래스명과 태그 구조를 동적으로 관리할 수 있도록 설계되었으며, **합성** 구조로 구현되어 각 UI 요소를 조합해 새로운 레이아웃을 쉽게 구성할 수 있게 개발했습니다.

### GridColumn 컴포넌트 예시

다음은 `GridColumn` 컴포넌트의 구현 예시입니다. 클래스명을 동적으로 생성하며, 태그 구조를 변경할 수 있어 다양한 레이아웃 요구사항에 대응할 수 있도록 설계되었습니다.

```jsx
type GridColumnProps = {
  rate: 4 | 6 | 12,
  children: React.ReactNode,
  medium?: 3,
  Tag?: "div" | "li",
};

const GridColumn = ({ rate, children, medium, Tag = "div" }: GridColumnProps) => {
  return (
    <Tag className={cx("l-grid__col", rate && `l-grid__col--${rate}`, medium && `medium-${medium}`)}>{children}</Tag>
  );
};
```

### 컬렉션 내 활용 예시

Wrapper 컴포넌트를 활용해 구현한 반복 구매 컬렉션의 일부 코드는 다음과 같습니다. `Grid`와 `GridRow`, `GridColumn` 등의 Wrapper 컴포넌트를 조합하여 가독성과 재사용성을 높인 구조로 개선할 수 있었습니다.

```tsx
const SearchOrderedProductCollection = ({ group }: SearchOrderedProductCollectionProps) => {
  //...
  return (
    <Grid>
      <GridRow dataUiType="Product_Reorder">
        <GridColumn rate={12}>
          <HeadLine>
            <HeadLineText>
              <HeadLineTitle>
                지난번 구매한<em> 이 상품</em> 어떠세요?
              </HeadLineTitle>
            </HeadLineText>
          </HeadLine>
          <Scroll>
          { /* ... */ }
        <GridColumn />
      <GridRow />
    <Grid />
  );
};
```

# 각 계층 간의 책임 명확히 하기

## 데이터의 흐름 및 변환

PC 코드는 **MobX**를 활용하여 `repository`, `model`, `store`, `view`로 구분되며 다음과 같은 역할을 수행합니다.

- **repository**: API 통신 및 데이터 원천 관리
- **model**: `Model`과 `ViewModel` 역할을 겸하며, 데이터 바인딩과 사용자 액션을 처리
- **store**: 상태 관리 및 데이터 로직 처리
- **view**: UI 렌더링 및 사용자 인터랙션 관리

예시로 검색 필터인 카테고리 필터를 보겠습니다.

### (1) repository

`FilterRepository`는 API 호출과 URL 정보를 관리하며, 데이터 원천 역할을 수행합니다.

```js
class FilterRepository {
  URL = process.env.ENV === "local" ? "/filters" : "/search/api/v1/tab-filter";
  constructor(url) {
    /*...*/
  }
  getFilters(params) {
    /*...*/
  }
  getFiltersByUrl(url) {
    /*...*/
  }
}
```

### (2) model

`CategoryFilterModel`은 `FilterModel`을 상속하며, 사용자 행동에 따른 `pushOrRemove`, `select`, `update` 등의 동작을 처리하고 스토어와 상호작용합니다.

```js
import { action, makeObservable } from "mobx";
import FilterModel from "./FilterModel";

class CategoryFilterModel extends FilterModel {
  constructor(store, data) {
    super(store, data);
    makeObservable(this, {
      update: action.bound,
      pushOrRemove: action.bound,
    });
  }
  pushOrRemove(isSelected) {
    /*...*/
  }
  select(isSelected) {
    /*...*/
  }
  update(isSelected) {
    /*...*/
  }
}
```

### (3) store

`FilterStore`는 필터 관련 비즈니스 로직과 상태를 관리합니다. 예를 들어, 카테고리 필터를 생성하고 관리하는 로직이 포함됩니다.

```js
class FilterStore {
  categoryFilter = {};
  // ...
  getCategoryFilterModel(item) {
    const model = new CategoryFilterModel(this, {
      ...item,
      filterType: "category",
      inputType: "radio",
      /*...*/
    });
    // ...
  }
}
```

### (4) view

`View`에서는 `rootStore`를 통해 `filterStore`에 접근하여 데이터를 렌더링합니다.
`rootStore`는 모든 스토어를 인스턴스화하고, 스토어 간 의존성을 관리합니다. 각 스토어는 `rootStore`를 통해 서로 협력합니다.

```jsx
const Filter = () => {
  const rootStore = useContext(StoreContext);
  const { filterStore: store, uiStore } = rootStore;

  return (
    <Box className="search_filter">
      <Text className="skip" text="검색필터" Tag="h3" />
      {!rootStore.isLoading && (
        <>
          /*...*/
          <FilterSortViewCategory store={store} uiStore={uiStore} />
          /*...*/
        </>
      )}
    </Box>
  );
};
```

### PC 코드 특징

PC 코드는 API에서 내려온 데이터를 `View`에 맞게 정제하거나 추가한 뒤, 스토어에 상태로 저장하여 사용합니다.
`View`로 데이터를 전달하기 전에, 노출 개수, 종류, 제한사항 등 모든 상태를 명시적으로 `Model`에 할당합니다.

초기 PC 코드는 각 계층의 책임이 구조적으로 잘 나뉘어져 있었습니다.
그러나 시간이 흐르며 여러 개발자들의 수정과 요구사항이 더해지면서 원래 의도와 달리 복잡해졌습니다.
특히 여러 곳에서 필터 키값이 반복적으로 추가되면서 데이터 흐름이 복잡해져 유지보수에 어려움을 겪게 되었습니다.

### 모바일 코드 특징

모바일 코드는 PC 코드와 비교했을 때 구조적으로 큰 차이는 없지만, `Model` 계층을 사용하지 않습니다. API에서 받은 데이터를 변경 없이 그대로 View로 전달하며, React Hooks를 활용해 컴포넌트 내부에서 데이터를 독립적으로 정제하거나 추가 처리합니다.

검색 서비스와 같이 변화가 빠르고 복잡성이 높은 서비스에서는 코드의 제거와 변경이 용이해야 합니다.
구조적으로 책임이 잘 나누어진 코드라도 흐름이 복잡하거나 단방향적이지 않으면 결국 레거시로 남게 됩니다.
충분한 시간이 주어진 이상적인 일정이라면 레거시를 제거할 수 있겠지만, 현실적으로 그렇지 않은 경우가 많았습니다.
결국, 레거시는 원래 의도와 다르게 다른 코드에 영향을 미치게 됩니다.

**검색 서비스에서 좋은 품질의 코드는 단순하고 독립적인 구조를 유지하는 것**입니다.

## 검색 컴포넌트의 의존성을 줄이자

검색 서비스에는 상품 정보를 조합하여 보여주는 다양한 컴포넌트가 사용됩니다. 하지만 단순히 데이터를 표시하는 역할을 넘어, 동영상 재생, 타이머(카운트다운), 팝업, 필터 등 데이터 조작과 복잡한 로직을 포함한 컴포넌트가 많았습니다. 하지만 이러한 로직이 컴포넌트 내부에 포함되어 있을 경우, 몇 가지 문제가 발생합니다.

- **재사용성 저하**

  - 특정 기능이 여러 영역으로 확장될 때, 로직이 컴포넌트 내부에 고정되어 있어 동일한 로직을 재사용하기 어렵습니다.

- **불필요한 의존성 증가**

  - 동일한 UI를 구현하지만 해당 로직이 필요하지 않은 컴포넌트도 불필요한 로직을 포함하게 되어 코드의 복잡성이 증가합니다.

- **유지보수 어려움**
  - 로직이 UI와 결합되어 있어 수정이나 테스트가 어려웠으며, 변경사항이 생길 때마다 컴포넌트를 직접 수정해야 합니다.

### Custom Hook으로 컴포넌트 의존성 줄이기

<figure style="width: 100%; margin-left: auto; margin-right: auto;">
    <a href="/files/post/2024-11-25-search-service/time-deal-hooks.png" class="mg-link">
        <img src="/files/post/2024-11-25-search-service/time-deal-hooks.png">
    </a>
</figure>

이 문제를 해결하기 위해 UI와 로직을 분리했습니다. 컴포넌트 내부에 포함된 데이터 처리 및 비즈니스 로직은 Custom Hook으로 분리하여 다음과 같은 이점을 얻을 수 있었습니다.

- 로직 재사용 가능
  - 여러 컴포넌트에서 동일한 로직을 필요로 할 때, Custom Hook을 호출함으로써 코드 중복을 줄이고 일관된 동작을 구현할 수 있었습니다.
- UI와 로직의 책임 분리
  - UI는 데이터를 표시하는 역할에만 집중하게 되었고, 로직은 Custom Hook에서 독립적으로 관리되었습니다.
- 테스트 용이성 증가
  - Custom Hook으로 분리된 로직은 독립적으로 테스트 가능하여 컴포넌트의 테스트 범위와 복잡성을 줄이고, 로직 자체의 신뢰성을 높일 수 있었습니다.

# 마치며

개발자는 항상 주어진 상황에서 최선의 코드를 작성하려 노력합니다.
'좋은 코드'의 정의는 팀이 맡은 도메인, 당시의 팀 상황, 그리고 프로젝트 환경에 따라 달라질 수 있습니다.
완성도 높은 구조를 설계하더라도, 백엔드, 기획, 디자인 등 다른 팀의 일정이나 요구 사항에 따라 정해둔 규칙을 벗어나야 하는 경우도 종종 발생합니다.

이러한 현실 속에서 중요한 것은 완벽한 ‘정답’을 찾는 것이 아니라, 맥락에 맞는 최선의 선택을 하는 것이라 생각합니다.
협의를 통해 팀 전체가 나아갈 방향을 정하고, 유연하게 대처하며 프로젝트를 완성해가는 것이 무엇보다도 중요합니다.

결국, 코드 설계에는 절대적인 정답이 없다는 사실을 항해 중 경험으로 배울 수 있었습니다.
우리가 할 수 있는 일은 복잡한 우주 한가운데에서 각자의 해답을 찾으려 노력하며, 끊임없이 나아가는 것입니다.
히치하이커처럼 여정을 즐기고, 필요한 순간엔 유연하게 방향을 수정하며, 결국 더 나은 코드를 위한 항해를 계속해나가는 것이 우리의 목표가 아닐까요?

"타월을 잊지 마세요."
