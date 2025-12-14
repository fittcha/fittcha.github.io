---
title: "이미지가 왜 안 떠? - 정적 파일 서빙 전략 알아보기"
date: 2025-12-14
categories: [Development, Infrastructure]
tags: [CDN, 정적파일, S3, CloudFront, 이미지서빙]
---

## 들어가며

블로그에 이미지를 넣었는데 404 에러가 떴습니다.

분명히 `/assets/img/posts/` 폴더에 이미지를 넣었고, 마크다운 경로도 맞게 썼고, git push도 했는데... 이미지만 안 뜨는 거예요.

```markdown
![Facade 패턴 개념도](/assets/img/posts/facade-pattern/facade-concept.png)
```

경로도 확인하고, 파일명 대소문자도 확인하고, 별짓을 다 해봤는데 안 됐어요. 근데 이상한 건 브라우저에서 이미지 URL을 직접 치면 잘 보인다는 거였습니다.

개발자 도구를 열어보니 404 에러 메시지에 **Netlify**라는 단어가 보였어요. 어... Netlify? 그게 뭐지? 어딘가의 이름인 것 같은데..

---

## 원인: CDN 설정이 범인이었다

제가 쓰는 Chirpy 테마의 `_config.yml`을 열어보니 이런 설정이 있었습니다.

```yaml
cdn: "https://chirpy-img.netlify.app"
```

이게 뭐냐면, 모든 이미지 경로 앞에 저 CDN 주소를 붙여버리는 설정이에요.

내가 쓴 경로:
```
/assets/img/posts/facade-pattern/facade-concept.png
```

실제로 요청되는 경로:
```
https://chirpy-img.netlify.app/assets/img/posts/facade-pattern/facade-concept.png
```

Chirpy 테마가 기본 제공하는 이미지들은 저 Netlify CDN에 올라가 있어서 잘 되는데, 내가 직접 올린 이미지는 거기 없으니까 404가 뜬 거였습니다.

해결은 간단했어요.

```yaml
cdn: ""
```

CDN 설정을 비워주니까 바로 해결됐습니다.

---

## 잠깐, CDN이 뭔데?

이 삽질을 하면서 "정적 파일을 어디서 서빙하느냐"가 생각보다 중요하다는 걸 깨달았어요. 그래서 이참에 정리해봤습니다.

**CDN(Content Delivery Network)**은 이미지, CSS, JS 같은 정적 파일을 전 세계 여러 서버에 분산 저장해두고, 사용자와 가장 가까운 서버에서 내려주는 방식이에요.

![CDN 개념도](/assets/img/posts/static-file-serving/cdn-server.png)

한국 사용자가 미국에 있는 원본 서버까지 갈 필요 없이, 서울에 있는 엣지 서버에서 바로 받으니까 빠른 거죠.

---

## 정적 파일 서빙 방식 비교

정적 파일을 서빙하는 방법은 크게 4가지가 있습니다.

### 1. 직접 서빙 (Same Origin)

웹 서버에서 직접 정적 파일을 내려주는 방식이에요. 지금 제 GitHub Pages가 이 방식입니다.

![direct origin](/assets/img/posts/static-file-serving/direct-origin.png)

**장점:**
- 설정이 단순함
- 추가 비용 없음
- 경로 관리가 쉬움

**단점:**
- 서버가 먼 지역 사용자는 느림
- 트래픽 많아지면 서버 부담

**적합한 경우:**
- 개인 블로그, 소규모 프로젝트
- 트래픽이 적은 서비스

### 2. CDN (Content Delivery Network)

전 세계 엣지 서버에 파일을 캐싱해두고 서빙하는 방식입니다.

![cdn](/assets/img/posts/static-file-serving/cdn.png)

대표 서비스: **Cloudflare**, **AWS CloudFront**, **Akamai**, **Fastly**

**장점:**
- 전 세계 어디서든 빠름
- 원본 서버 부하 감소
- DDoS 방어 기능도 있음

**단점:**
- 캐시 무효화(purge)가 즉시 안 될 수 있음
- 비용 발생 (트래픽 기반 과금)
- 설정 복잡도 증가

**적합한 경우:**
- 글로벌 서비스
- 트래픽 많은 서비스
- 이미지/동영상 많이 쓰는 서비스

### 3. Object Storage (S3, GCS)

파일을 별도 저장소에 올려두고 URL로 접근하는 방식입니다.

![object storage](/assets/img/posts/static-file-serving/object-storage.png)

대표 서비스: **AWS S3**, **Google Cloud Storage**, **Azure Blob**

**장점:**
- 저장 용량 거의 무제한
- 웹 서버와 분리되어 관리 편함
- 비용 효율적 (저장 + 전송량 기반)

**단점:**
- CDN 없이 쓰면 지역별 속도 차이
- 버킷 설정, 권한 관리 필요
- 파일 업로드 로직 따로 구현해야 함

**적합한 경우:**
- 사용자 업로드 파일 관리
- 대용량 파일 저장
- 백업/아카이브

### 4. S3 + CloudFront (하이브리드)

실무에서 가장 많이 쓰는 조합입니다. S3에 파일을 저장하고, CloudFront로 전 세계에 캐싱해서 서빙해요.

![s3 + cloudfront](/assets/img/posts/static-file-serving/s3-cloudfront.png)

**장점:**
- 저장은 S3에서 (저렴하고 무제한)
- 서빙은 CloudFront에서 (빠르고 안정적)
- 원본(S3)은 비공개로 두고 CloudFront만 public 가능

**단점:**
- 설정할 게 많음 (S3 버킷, CloudFront 배포, 권한 등)
- 두 서비스 비용 합산

**적합한 경우:**
- 상용 서비스 대부분
- 이커머스 상품 이미지
- 미디어 플랫폼

---

## 비교 정리

| 방식 | 속도 | 비용 | 복잡도 | 적합한 상황 |
|------|------|------|--------|-------------|
| 직접 서빙 | 보통 | 무료/저렴 | 낮음 | 개인 블로그, 소규모 |
| CDN만 | 빠름 | 중간 | 중간 | 글로벌, 정적 사이트 |
| S3만 | 보통 | 저렴 | 중간 | 파일 저장 위주 |
| S3 + CloudFront | 빠름 | 중간 | 높음 | 상용 서비스 |

---

## 실무에서는 어떻게 쓸까?

### 이커머스 상품 이미지

대부분 **S3 + CloudFront** 조합을 씁니다.

![s3 + cloudfront hands-on](/assets/img/posts/static-file-serving/s3-cloudfront-hands-on.png)

상품 이미지가 몇십만 개가 되면 웹 서버에서 직접 관리하기 힘들어요. S3에 올려두고 CloudFront URL로 서빙하는 게 일반적입니다.

### 사용자 업로드 파일

프로필 사진, 리뷰 이미지 같은 건 **S3 직접 업로드** 방식을 많이 써요.

![s3 direct upload](/assets/img/posts/static-file-serving/s3-direct-upload.png)

서버를 거치지 않고 S3에 바로 올리면 서버 부하를 줄일 수 있습니다.

### 정적 웹사이트

개인 블로그나 문서 사이트는 **GitHub Pages + Cloudflare** 조합도 괜찮아요. 무료인데 꽤 빠릅니다.

---

## 캐시 주의점

CDN을 쓸 때 가장 흔히 겪는 문제가 **캐시 무효화**예요.

이미지를 수정해서 다시 올렸는데, CDN이 예전 이미지를 계속 내려주는 경우가 있어요. 캐시가 만료되기 전까진 새 이미지가 안 보이는 거죠.

**해결 방법:**

1. **파일명에 버전/해시 붙이기**
   ```
   logo.png → logo.v2.png
   logo.png → logo.a3f2b1c.png
   ```

2. **쿼리스트링 버전**
   ```
   logo.png?v=2
   logo.png?t=1702451234
   ```

3. **수동 캐시 퍼지**
   - CloudFront: Invalidation 요청
   - Cloudflare: Purge Cache

실무에서는 빌드할 때 파일명에 해시를 자동으로 붙이는 방식을 많이 씁니다.

---

## 정리

이미지 404 에러 하나 잡으려다가 정적 파일 서빙에 대해 정리하게 됐네요.

핵심만 요약하면:

- **직접 서빙**: 간단하지만 규모 커지면 한계
- **CDN**: 빠르지만 캐시 관리 필요
- **S3**: 저장에 최적화, 단독으론 느릴 수 있음
- **S3 + CloudFront**: 실무 표준 조합

개인 블로그는 GitHub Pages로 충분하고, 상용 서비스라면 S3 + CloudFront 조합을 추천합니다.

다음 편에서는 이미지 최적화랑 이커머스에서 상품 이미지 처리하는 법을 다뤄볼게요!

