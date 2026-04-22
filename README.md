# G_every_time_clowl

에브리타임(EVERYTIME) 커뮤니티에서 게시글/댓글 데이터를 수집하고,  
수집한 텍스트를 혐오표현 라벨로 분류해 **온라인 커뮤니티 텍스트의 분위기와 상호작용 패턴**을 확인한 프로젝트입니다.

단순 크롤링에서 끝내지 않고, 수집 데이터에 NLP 분류 모델을 적용해  
`title(제목)`, `article(본문)`, `reple(댓글)` 각각의 언어적 특성을 비교하는 흐름으로 구성했습니다.

---

## 1) 프로젝트 배경과 목적

커뮤니티 데이터는 실제 사용자 반응이 빠르게 축적되는 텍스트 자원입니다.  
이 프로젝트는 다음 질문에 답하기 위해 진행했습니다.

- 핫게시물 기준으로 수집했을 때, 어떤 시점에 게시글이 많이 생성되는가?
- 게시글의 길이와 댓글 활동성은 어떤 분포를 보이는가?
- 제목/본문/댓글 중 어느 구간에서 혐오표현 라벨 비중이 상대적으로 높게 나타나는가?

즉, **데이터 수집(크롤링) + 데이터 분석(통계) + 텍스트 라벨링(NLP)**을 하나의 파이프라인으로 구현하는 데 초점을 맞췄습니다.

---

## 2) 기술 스택

- **Crawling**: `Selenium`, `BeautifulSoup`, `requests`
- **Data Handling**: `pandas`, `openpyxl`
- **Modeling / NLP**: `transformers`, `torch`, `tensorflow`, `keras`
- **Environment**: Jupyter Notebook / Google Colab

---

## 3) 데이터 수집 파이프라인

크롤링 로직은 `every_time_clow.ipynb`에 구현되어 있습니다.

1. `selenium.webdriver`로 에브리타임 로그인 세션 생성
2. `https://everytime.kr/hotarticle/p/{page}` 형태의 목록 페이지 순회
3. 게시글 상세 URL 수집 후 각 페이지에 재접속
4. 아래 필드를 파싱해 DataFrame에 적재
   - `time`: 작성일
   - `URL`: 게시글 링크
   - `title`: 제목
   - `article`: 본문
   - `reple`: 댓글 텍스트(줄바꿈 기반 누적)
5. URL 중복 체크 후 `clow_data_final.xlsx`로 저장

핵심은 **목록 페이지 수집 + 상세 페이지 재파싱 + 중복 제거** 구조입니다.

---

## 4) 데이터셋 개요

`clow_data_final.xlsx` 기준:

- 수집 게시글: **408건**
- 고유 URL: **407건** (중복 1건)
- 수집 기간: **2024-09-01 ~ 2024-10-31**
- 월별 분포
  - 2024-09: **157건**
  - 2024-10: **251건**

본문 길이(문자 수) 분포:

- 평균: **230.58**
- 중앙값: **65**
- IQR(1~3사분위): **18 ~ 215**
- 최대: **10,000**

댓글 활동성:

- 전체 댓글 수: **7,508**
- 게시글당 평균 댓글: **18.4**
- 댓글 1개 이상 게시글 비율: **98.28% (401/408)**

상호작용(댓글) 상위 게시글:

1. 과잠이 궁금한 사람들을 위하여... (24학번 기준 작성) - 135
2. 지거국 비동일계로 편입 간절한 1학년들 필독 - 120
3. 담배피는 사람들아 - 107
4. 진주 토박이가 알려주는 진주맛집 - 103
5. 스팅어 차량 조심하세요! 🚘🚗🚘🚗 - 100

---

## 5) 라벨링(모델 적용) 설명

라벨링/분석 로직은 `final.ipynb`에서 수행했습니다.

### 사용 모델

- **주요 적용 모델**: `smilegate-ai/kor_unsmile`
  - Hugging Face `transformers` 기반 한국어 혐오표현 분류 모델
  - `TextClassificationPipeline` + `sigmoid` 스코어를 사용해 문장 단위 라벨 추론
  - 모델 링크: [https://huggingface.co/smilegate-ai/kor_unsmile](https://huggingface.co/smilegate-ai/kor_unsmile)

### 모델이 학습한 데이터셋

- **Korean UnSmile Dataset**(Smilegate AI에서 공개한 한국어 혐오표현 분류 데이터셋)을 기반으로 학습된 모델을 사용했습니다.
- 데이터셋 링크: [https://github.com/smilegate-ai/korean_unsmile_dataset](https://github.com/smilegate-ai/korean_unsmile_dataset)
- 데이터셋은 온라인 텍스트를 다중 라벨 관점에서 분류할 수 있도록 구성되어 있으며, 대표적으로 아래 범주의 라벨을 포함합니다.
  - `clean`(비혐오)
  - `악플/욕설`
  - `여성/가족`, `남성`
  - `인종/국적`, `지역`
  - `연령`, `종교`, `성소수자`, `기타 혐오`
- 본 프로젝트에서는 이 사전학습/파인튜닝된 공개 모델을 그대로 활용해, 수집한 제목/본문/댓글 텍스트를 동일한 라벨 체계로 매핑해 분포를 확인했습니다.

### 라벨링 방식

- 입력 대상: `title`, `article`, `reple` 컬럼
- 텍스트를 줄바꿈 단위로 분할한 뒤 문장별 분류
- 각 텍스트 구간의 라벨 결과를 누적해 `Counter`로 분포 확인

### 관찰 결과 (노트북 출력값 기준)

- 제목(408개)
  - `clean`: 353 (**86.52%**)
  - `악플/욕설`: 44 (**10.78%**)
- 본문(408개)
  - `clean`: 313 (**76.72%**)
  - `악플/욕설`: 70 (**17.16%**)
- 댓글(4,050개 라벨)
  - `clean`: 3,526 (**87.06%**)
  - `악플/욕설`: 428 (**10.57%**)

해석 관점:

- 제목보다 본문에서 `악플/욕설` 라벨 비중이 더 높게 나타났습니다.
- 댓글은 절대량이 커서 커뮤니티 반응의 세부 분포를 보기에 유용했습니다.
- 댓글 총량(7,508)과 라벨 집계량(4,050)은 전처리/분할 로직 차이로 다를 수 있습니다.

---

## 6) 프로젝트 파일 구조

- `every_time_clow.ipynb`: 에브리타임 크롤링 로직
- `clow_data_final.xlsx`: 크롤링 결과 데이터셋
- `final.ipynb`: 라벨링 및 통계 확인 노트북

---

## 7) 회고 및 개선 아이디어

- 시간 정보가 일(day) 단위로 정규화되어 있어 시(hour) 단위 분석이 제한적이었습니다.
- 댓글 라벨 집계 기준(문장 분할/길이 제한)을 명시적으로 고정하면 재현성이 더 좋아집니다.
- 향후 개선:
  - 수집 대상 게시판 확장(핫게시물 외 일반 게시판)
  - 라벨별 샘플 검수(정성 평가) 추가
  - 시각화 대시보드(월별/카테고리별 라벨 트렌드) 구축

---

## 8) 실행 및 재현 시 유의사항

- 로그인 계정 정보는 코드에 직접 저장하지 말고, 로컬 환경 변수나 입력 방식으로 주입하는 것을 권장합니다.
- 사이트 이용약관/robots 정책/요청 빈도 제한을 준수해야 합니다.
- `final.ipynb`는 Colab 경로(`/content/drive/...`) 기반 코드가 포함되어 있어, 로컬 실행 시 파일 경로 수정이 필요합니다.

---

## 9) 참고 링크

- Korean UnSmile Dataset (GitHub): [https://github.com/smilegate-ai/korean_unsmile_dataset](https://github.com/smilegate-ai/korean_unsmile_dataset)
- `smilegate-ai/kor_unsmile` 모델 (Hugging Face): [https://huggingface.co/smilegate-ai/kor_unsmile](https://huggingface.co/smilegate-ai/kor_unsmile)

<sub>※ 라이선스 참고: UnSmile 데이터셋(CC-BY-NC-ND 4.0), baseline 코드/모델(Apache 2.0)</sub>
