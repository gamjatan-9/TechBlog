---
title: "Pagination"
date: "2024-02-22"
tags: ["스터디", "react-query", "lighthouse", "이미지 최적화"]
desc: "구현 및 최적화 스터디 1"
---

### ❇️ 결과물
![alt text](/images/post-1/1.png)

### ❇️ Lighthouse 성능 측정
![alt text](/images/post-1/2.png)
![alt text](/images/post-1/3.png)

### ❇️ Tanstack Query - 비동기 API 호출 

```
// src/hooks/useFetchData.ts

import { useQuery } from '@tanstack/react-query';

async function fetchData (page: number, limit: number = 12) {
    // sessionStorage 데이터 존재 유무 확인.
    const cachedData = sessionStorage.getItem(`catImages-page-${page}`);
    if (cachedData) return JSON.parse(cachedData);

  const response = await fetch(`https://api.thecatapi.com/v1/images/search?limit=${limit}&page=${page}`, {
    headers: { 'x-api-key': 'api-key' },
  });
  if (!response.ok) throw new Error('Network response was not ok');

  const data = await response.json();
  // 데이터를 세션 스토리지에 저장
  sessionStorage.setItem(`catImages-page-${page}`, JSON.stringify(data));
  return data;
};

export function useFetchData (page: number) {
  return useQuery({
    queryKey: ['catImages', page],
    queryFn: () => fetchData(page),
    staleTime: 5 * 60 * 1000, 
    gcTime: 24 * 60 * 60 * 1000,
  });
};
```

### ❇️ Pagination

```
// src/components/Pagination.tsx

function Pagination() {
  const { currentPage, setCurrentPage, totalPages } = useImages();
  //시작 페이지 계산 (1, 6, 11, 16)
  const startPage = Math.floor((currentPage - 1) / 5) * 5 + 1; 

  return (
    <Pages>
      <Button onClick={() => setCurrentPage(1)} disabled={currentPage === 1}>
        {"<<"}
      </Button>
      <Button onClick={() => setCurrentPage(currentPage - 1)} disabled={currentPage === 1}>
        {"<"}
      </Button>
      {Array.from({ length: Math.min(5, totalPages - startPage + 1) }, (_, index) => (
        <Button
          key={startPage + index}
          disabled={currentPage === startPage + index}
          onClick={() => setCurrentPage(startPage + index)}
        >
          {startPage + index}
        </Button>
      ))}
      <Button onClick={() => setCurrentPage(currentPage + 1)} disabled={currentPage === totalPages}>
        {">"}
      </Button>
      <Button onClick={() => setCurrentPage(totalPages)} disabled={currentPage === totalPages}>
        {">>"}
      </Button>
    </Pages>
  );
}

```

### ❇️ 이미지 최적화

1. Lazy Loading

```
// src/components/Images.tsx
const ImageLoader = ({ src, alt }: { src: string; alt: string }) => {
  const [loaded, setLoaded] = useState(false);

  return (
    <ImageContainer>
      {!loaded && <Skeleton />}
      <StyledImage
        src={src}
        alt={alt}
        style={{ opacity: loaded ? 1 : 0 }}
        onLoad={() => setLoaded(true)}
        loading="lazy"
      />
    </ImageContainer>
  );
};
```

2. UX 개선 - CSS opacity, Skeleton UI

```
// src/components/Images.tsx

function Images() {
  const { images, isLoading, error } = useImages();
  if (error) return <div>Error: {error.message}</div>;

  return (
    <GridContainer>
      {isLoading
        ? Array.from({ length: 12 }).map((_, index) => (
            <ImageContainer key={index}>
              <Skeleton />
            </ImageContainer>
          ))
        : images?.map((image) => (
            <ImageLoader key={image.id} src={image.url} alt="cat" />
          ))}
    </GridContainer>
  );
};

```

### ❇️ 느낀점

이번 스터디를 통해 리액트 성능 최적화를 깊게 고민해보며 코드를 작성해 보았다.
최적화는 사용자 경험, 비용 효율성, 검색 엔진 최적화(SEO), 그리고 기술적 지속 가능성을 포함하여 여러 가지 이유로 중요하다. 이전에는 그저 구현에만 급급하고 세부적인 요소까지는 생각을 하지 못했는데 이번 기회로 좀 더 좋은 코드를 작성할 수 있었다.

이미지 최적화에 대해서 찾아본 결과 이미지 크기가 가장 중요한 부분으로 보인다. 이미지 크기가 큰 경우 웹 페이지의 로딩 시간이 길어져 사용자 경험에 부정적인 영향을 미칠 수 있다. 그러므로 필요에 따라 이미지 크기를 줄여 로딩 시간을 최소화해야 한다. 일반적으로 같은 이미지에 대해 여러 크기의 이미지를 준비하여, 필요에 따라 이미지 크기를 선택하여 사용한다. 예를 들어, 화질이 높을 필요가 없는 썸네일에는 작은 크기의 이미지를 사용하고, 이미지의 세부 사항이 중요한 경우에는 큰 이미지를 사용한다.

하지만, 이 프로젝트에서는 Open API를 사용하여 이미지를 불러오기 때문에 이미지 크기를 자유롭게 조절할 수 없다는 한계가 있어 아쉬웠다. 이미지 형식을 PNG나 JPEG에서 로딩 시간이 더 짧은 WebP나 WebM 같은 형식으로 변경하는 것이 로딩 시간을 단축시킬 수 있다. API 이미지의 확장자를 바꿔보는 시도를 해봤지만 이 역시 Open API의 한계로 불가능했다.

팀원들의 코드를 보며 React의 Suspense 기능을 활용하면 좋을 것 같다고 생각했다. <Suspense>는 children 이 로딩되기 전에 fallback을 보여줄 수 있다. fallback에는 skeleton UI를 적용시키면 된다.
React Query와 Suspense를 함께 사용할 때, 선언형 프로그래밍이 가능하다. React Query는 데이터의 로딩 상태, 캐싱, 업데이트, 오류 처리 등을 내부적으로 관리한다. Suspense와 함께 사용하면, 데이터가 준비될 때까지 자동으로 대기하고, 준비되면 컴포넌트를 렌더링하는 방식으로 작동한다. 따라서 코드의 복잡성을 줄이고 가독성을 향상시킨다는 장점이 있다.

Skeleton UI를 적용시키는 부분에서 Skeleton 위에 이미지가 그대로 나타나야 하는데 로딩되는 과정에서 위치가 이리저리 움직이는 현상이 나타났다. 이 부분에 대해서는 처음에는 css의 visibility 속성을 사용했다. 로드가 완료되면 이미지가 나타나게 설정해주었다. 그런데 문제가 있었다. skeleton -> skeleton 사라짐 -> 이미지 나타남 이런 과정이 되었다. skeleton이 사라지는 것을 막기 위해 visibility 속성 대신 opacity 속성을 사용했다. 이렇게 변경하니 문제가 해결되었다.

<br />
<br />
<br />
참고: [명령형 vs 선언형 프로그래밍](https://lasbe.tistory.com/160)