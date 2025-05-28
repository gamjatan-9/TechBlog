---
title: "useInfiniteQuery로 무한스크롤 구현, Masonry 레이아웃"
date: "2024-03-07"
tags: ["스터디", "react-query", "무한스크롤"]
desc: "구현 및 최적화 스터디 2"
---

## ❇️ 결과물
![alt text](/images/post-2/1.png)
![alt text](/images/post-2/2.png)

## ❇️ React Query - useInfiniteQuery

[thecatapi](https://thecatapi.com/)
에서 고양이 사진을 무한으로 불러오기 위해 react-query에서 제공하는 useInfiniteQuery 훅을 사용했습니다.

```
//UseFetchInfiniteData.ts

import { useInfiniteQuery } from '@tanstack/react-query';

async function fetchInfiniteData({ pageParam = 1 }) {
  const limit = 10; // 한 번에 불러올 데이터 수
  const response = await fetch(`https://api.thecatapi.com/v1/images/search?limit=${limit}&page=${pageParam}`, {
    headers: { 'x-api-key': `${process.env.REACT_APP_REST_API_KEY}` },
  });

  if (!response.ok) throw new Error('api error');

  const data = await response.json();
  const nextPage = data.length === limit ? pageParam + 1 : undefined; //  다음 페이지 여부 판단
  return { data, nextPage };
};

export function useFetchInfiniteData() {
  return useInfiniteQuery({
    queryKey: ["catImages"],
    queryFn: fetchInfiniteData,
    initialPageParam: 1,
    getNextPageParam: (lastPageParam) => lastPageParam.nextPage,
  });
}
```

**useInfiniteQuery 반환값**

- queryKey: 고유 식별자
- queryFn: 데이터를 불러올 때 실행할 함수, 앞서 정의한 'fetchInfiniteData'함수를 지정
- initialPageParam: 최초 페이지 번호로, 1로 설정
- getNextPageParam: 다음 페이지를 불러올 때 사용할 페이지 번호를 결정하는 역할. 이 함수는 마지막으로 불러온 페이지의 정보를 인자로받아, 해당 페이지에서 정의한 'nextPage'값을 반환

### ❇️ 전역 상태관리 - Context API 

고양이 이미지 데이터와 무한 스크롤 데이터를 전역 상태로 관리합니다.

```
//CatsContext.tsx
import React, { createContext, ReactNode } from 'react';
import { useFetchInfiniteData } from '../hooks/useFetchInfiniteData';
import { Cat } from '../models/cat';

// context에 전달될 값
interface CatsContextType {
  cats: Cat[]; 
  fetchNextPage: () => void; // 다음 페이지 데이터 불러오기 함수
  hasNextPage: boolean | undefined; // 다음 페이지 존재 여부
  isFetchingNextPage: boolean; // 데이터 불러오는 중인지 여부
  error: Error | null; 
}

export const CatsContext = createContext<CatsContextType | null>(null);

export const CatsProvider = ({ children }: { children: ReactNode }) => {
  const {
    data,
    error,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useFetchInfiniteData();

  const cat: Cat[] = data?.pages
    .flatMap(page =>
      page.data
        //.filter((cat: any) => cat.breeds.length > 0)
        .map((cat: any) => ({
          id: cat.id,
          url: cat.url,
          width: cat.width,
          height: cat.height,
          breeds: cat.breeds,
        }))
    ) ?? [];

  return (
    <CatsContext.Provider value={{
      cats: cat,
      fetchNextPage,
      hasNextPage,
      isFetchingNextPage,
      error
    }}>
      {children}
    </CatsContext.Provider>
  );
};
```

### ❇️ Masonry 레이아웃 - CSS Grid, Skeleton UI

```
import React, { useEffect, useRef, useState } from 'react';
import styled, { keyframes } from 'styled-components';
import { useCats } from '../../hooks/useCats';
import Modal from '../../components/Modal';
import { Cat } from '../../models/cat';

const GridContainer = styled.div`
  display: grid;
  padding: 10px;
  grid-template-columns: repeat(auto-fill, minmax(300px,1fr));
`;

const ImageContainer = styled.div<{ photoSpan: number }>`
  cursor: pointer;
  grid-row: span ${props => props.photoSpan};
  padding: 10px;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: opacity 0.5s ease-in-out;
    border-radius: 15px;
  }
`;

const BreedContainer = styled.div`
  display: flex;
  width: inherit;
  align-items: center;
`;

const loading = keyframes`
  0% {
    background-position: -468px 0
  }
  100% {
    background-position: 468px 0
  }
`

const Skeleton = styled.div<{ aspectRatio: string }>`
  width: 100%;
  border-radius: 15px;
  background: linear-gradient(90deg, #EAEAEA 25%, #eee 50%, #EAEAEA 75%);
  animation: ${loading} 2s ease-in-out infinite;
  padding-top: ${props => props.aspectRatio};
`;

const ImageLoader = React.memo(({ id, url, width, height, breeds }: Cat) => {
  const [loaded, setLoaded] = useState(false);
  const aspectRatio = `${(height / width) * 100}%`;
  const photoSpan = Math.ceil((height / width) * 200 / 10);
  const [modalOpen, setModalOpen] = useState(false); 

  return (
    <>
      <ImageContainer photoSpan={photoSpan}>
        {!loaded && <Skeleton aspectRatio={aspectRatio} />}
        <img
          alt="cat"
          src={url}
          style={{ opacity: loaded ? 1 : 0 }}
          onLoad={() => setLoaded(true)}
          onClick={() => setModalOpen(true)}
        />
        <Modal modalOpen={modalOpen} modalClose={() => setModalOpen(false)}>
          <img src={url} alt={id} style={{ maxWidth: '100%', maxHeight: '70dvh' }} />
          <BreedContainer>
            {breeds?.map(breed => <div key={breed.id}>{breed.name}</div>)}
          </BreedContainer>
        </Modal>
      </ImageContainer>
      <BreedContainer>

      </BreedContainer>
    </>
  );
});

```

**Masonry 레이아웃 구현**

- GridContainer 컴포넌트: grid-template-columns 속성을 사용하여 열의 최소 너비를 300px, 열 너비가 컨텐츠에 자동으로 맞추도록 설정합니다.

- ImageContainer 컴포넌트: 'photoSpan' prop은 이미지의 물리적 크기에 따라 동적으로 그리드 레이아웃 내에서 이미지의 크기를 조정합니다. prop을 전달하여 해당 컴포넌트가 CSS 그리드 내에서 차지하는 행의 수를 결정합니다.

 

**스켈레톤 UI 구현**

- useState를 사용하여 이미지의 로딩 상태를 관리합니다. 이미지가 로드되기 전에는 스켈레톤 UI를 보여줍니다.

- 이미지가 로드되면 onLoad 이벤트 핸들러를 통해 로딩 상태를 업데이트하고, 이미지를 표시합니다.

- 이미지의 가로세로 비율(aspectRatio)을 계산하여 스켈레톤의 크기를 동적으로 조정합니다.

### ❇️ 이미지 무한 스크롤로 표시하기 - IntersectionObserver

Intersection Observer API를 사용하여 사용자가 스크롤하여 페이지 하단에 도달했을 때 추가 이미지를 불러오는 무한 스크롤 기능을 구현했습니다. 'loader' ref를 통해 관찰할 DOM 요소를 지정하고, 해당 요소가 화면에 보일 때 추가 데이터를 불러옵니다.



```
function Images() {
  const { cats, fetchNextPage, hasNextPage, isFetchingNextPage, error } = useCats();
  const loader = useRef(null);

  useEffect(() => {
    const io = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting && hasNextPage) {
        fetchNextPage();
      }
    }, { threshold: 0.5 }); 
    if (loader.current) {
      io.observe(loader.current);
    }
    return () => io.disconnect();
  }, [fetchNextPage, hasNextPage, isFetchingNextPage]);

  if (error) return <div>Error: {error.message}</div>;

  return (
    <GridContainer>
      {cats?.map((cat) => (
        <ImageLoader 
          id={cat.id} 
          url={cat.url} 
          width={cat.width} 
          height={cat.height} 
          breeds={cat.breeds} />
      ))}
      <div ref={loader} style={{height: "100px"}}/>
    </GridContainer>
  );
}

export default Images;

```

<br />

### ❇️ 느낀점

useInfiniteQuery를 사용하면서 react query에 대한 이해가 깊어졌다. 또한 처음 봤을 때는 생소했던 공식 문서의 내용들이 이제는 익숙하게 읽혀졌다. 추가로 다른 훅을 사용해야 할 때에도 공식 문서를 보면서 수월하게 적용해볼 수 있을 것 같다.

<br />

masonry 레이아웃을 구현하는데에 어려움이 있었다. 이 레이아웃을 선택한 이유는 핀터레스트 사이트의 이미지 레이아웃을 구현해보는 것이 목표였기 때문이다. 처음에는 이 레이아웃의 명칭도 몰랐는데 검색을 하는 과정에서 masonry라고 불린다는 것을 알게 되었다.
아래는 초기 코드이다.

```
const GridContainer = styled.div`
  column-count: 3;
  column-gap: 20px; 
  width: 100%;
  break-inside: avoid; 
`;

const ImageContainer = styled.div`
  break-inside: avoid;
  margin-bottom: 20px;
`;
```

단순히 column-count와 gap으로만 그리드를 조절했다. 여기에서 문제는 스크롤을 내리고 다음 이미지가 불러와질 때, 높이값이 재조정되면서 이미 로드된 이미지가 고정되어 있지 않고, 움직인다는 것이다.
![alt text](/images/post-2/3.mp4)

스터디원의 코드와 추가로 grid 관련 글을 보고 코드를 수정했다. 위에서 수정된 코드를 볼 수 있다. 스켈레톤도 자연스럽게 잘 나오는 것을 볼 수 있다.

![alt text](/images/post-2/4.mp4)

개선해볼 사항은 성능이다. openAPI를 사용하고 있어서 이미지를 불러오는데 제한이 있기는 하지만 조금 더 최적화 요소를 찾아보면 좋을 것 같다.

![alt text](/images/post-2/5.png)
![alt text](/images/post-2/6.png)