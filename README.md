# 🎬 Movieplex

**영화 예매 · 리뷰 커뮤니티 통합 플랫폼**

TMDB Open API를 연동한 실시간 영화/TV 정보 제공부터, 좌석 선택 기반 예매, PG 결제, 실시간 채팅, 리뷰·좋아요·쿠폰 시스템까지 하나의 서비스 안에서 구현한 Spring 기반 웹 애플리케이션입니다.

> K-Digital Training(클라우드(AWS) 활용 자바/스프링 개발 부트캠프) 팀 프로젝트로 진행했으며,
> 4주간(2025.03.17 ~ 2025.04.14) 3인 팀이 기획부터 배포까지 전 과정을 함께 진행했습니다.

<br>

## 목차

1. [프로젝트 소개](#1-프로젝트-소개)
2. [주요 기능](#2-주요-기능)
3. [기술 스택](#3-기술-스택)
4. [아키텍처](#4-아키텍처)
5. [ERD](#5-erd)
6. [팀원 및 담당 역할](#6-팀원-및-담당-역할)
7. [트러블 슈팅](#7-트러블-슈팅)
8. [실행 화면](#8-실행-화면)

<br>

## 1. 프로젝트 소개

Movieplex는 **영화 예매 서비스**와 **영화/TV 리뷰 커뮤니티(ReviewNest)**를 함께 제공하는 플랫폼입니다.

- 상영작 정보는 자체 DB가 아닌 **TMDB(The Movie Database) Open API**를 실시간으로 호출해 최신 데이터를 유지합니다.
- 예매는 상영관 좌석 배치를 시각적으로 선택하는 방식으로 구현했고, **아임포트(I'mport) PG**를 연동해 실제 카드 결제 흐름(결제 → 검증 → 취소)을 처리합니다.
- 커뮤니티 영역에서는 리뷰 작성/댓글/좋아요, 실시간 1:1 채팅, 공지사항·FAQ·Q&A 게시판, 관리자 페이지를 제공합니다.

<br>

## 2. 주요 기능

### 🎫 영화 예매

**예매 플로우**
1. `TMDB API` 연동으로 실시간 상영작 목록을 불러와 예매 화면(`booking`) 노출
2. 상영관 선택 → 상영관별 예매 가능 좌석 현황 조회(`getSeats`) → 좌석 배치도(`seatBook`) 렌더링
3. 결제 수단 선택(카드 / 무통장) → 보유 쿠폰 중 **미사용 쿠폰만 필터링**해 노출 → 결제 페이지 이동
4. 결제 완료 후 예매 완료 페이지에서 상영관 정보·좌석·최종 결제 금액 확인

**결제 처리 (아임포트 연동)**
- 카드 결제는 클라이언트에서 결제 완료 후 전달받은 `imp_uid`로 **아임포트 서버에 재조회하여 실결제 금액과 요청 금액을 서버가 직접 대조**하는 방식으로 검증(`checkAmounts`) — 클라이언트 측 결제 금액 조작을 막기 위한 서버 사이드 검증 로직
- 금액이 일치하지 않으면 결제를 즉시 취소 처리(`cancelPaymentByImpUid`)하고 예매를 생성하지 않음
- 무통장(계좌이체) 예매는 결제 검증 없이 예약 건만 먼저 생성되는 별도 플로우(`movieBookBankBook`)로 분리 구현
- 예매 취소 시 저장된 결제 고유번호(`merchant_uid`)로 아임포트 환불 API를 호출하는 취소/환불 로직 별도 구현
- 결제 성공 시 좌석 정보 저장 → 결제 내역 저장 → 사용한 쿠폰 처리(회원/쿠폰 테이블 동시 갱신)까지 이어지는 흐름으로 처리

**좌석/상영관**
- 좌석은 리스트(`List<String>`) 형태로 상영관별 예약 좌석을 관리, 예매 시 좌석 목록을 일괄 등록
- 상영관 등록/삭제, 날짜별 상영 스케줄 조회는 관리자 페이지의 상영관 관리 기능과 연동

### 📝 ReviewNest (리뷰 커뮤니티)
- 영화/TV 콘텐츠별 리뷰 작성, 댓글, 대댓글
- 콘텐츠 좋아요(찜) / 리뷰 좋아요 기능
- 마이페이지에서 내가 쓴 리뷰·찜한 콘텐츠 모아보기

### 💬 실시간 채팅
- Spring WebSocket + SockJS 기반 실시간 1:1 채팅
- HttpSession 인터셉터로 채팅 세션 관리, 채팅방 목록/상세 조회

### 👤 사용자
- 일반 로그인/회원가입 + **카카오 소셜 로그인** 연동
- 이메일 인증 발송(SMTP), 마이페이지, 회원 정보 관리

### 🛠 관리자 페이지

**접근 제어**
- `/admin/*` 경로 전체에 `HandlerInterceptor`(`AdminCheckInterceptor`)를 등록해, 컨트롤러 진입 전 단계에서 권한을 검증
- 세션에 로그인 정보가 없으면 로그인 페이지로, 로그인은 되어 있으나 관리자 권한이 없으면 접근 거부 메시지와 함께 메인으로 리다이렉트 — 개별 컨트롤러마다 권한 체크 코드를 넣지 않고 **인터셉터 한 곳에서 관리자 기능 전체를 일괄 방어**하도록 설계

**회원 관리**
- 페이징 처리된 전체 회원 목록 조회 및 회원 상세 조회
- 회원 등급(관리자 권한 부여/해제), 결제 권한(정지/해제) 변경, 회원 강제 탈퇴 처리
- 각 처리 결과는 AJAX 공통 응답(`commons/ajax`)으로 반환해 목록 화면 갱신

**상영관 관리**
- 상영관 등록 시 TMDB 연동 영화 목록에서 상영작을 선택해 상영관과 매핑
- 날짜별 상영 스케줄 조회, 상영관 등록/삭제 CRUD

**쿠폰 발급**
- 관리자가 쿠폰을 생성·발급하면, 사용자 예매 화면의 보유 쿠폰 목록에 반영되는 구조

**게시판 관리**
- 공지사항/FAQ/Q&A 게시글 등록·수정·삭제 및 파일 첨부 관리

### 📋 게시판 / 파일
- 공지사항, FAQ, Q&A 게시판 CRUD 및 파일 첨부·다운로드 공통 모듈

<br>

## 3. 기술 스택

| 구분 | 내용 |
|---|---|
| **Language** | Java 8 |
| **Framework** | Spring Framework 4.3 (MVC), Spring WebSocket |
| **Persistence** | MyBatis 3, Oracle Database (ojdbc11) |
| **View** | JSP, JSTL, Bootstrap, jQuery, Summernote(에디터) |
| **외부 연동** | TMDB Open API(영화/TV 데이터), 아임포트(I'mport) 결제, 카카오 로그인 API, JavaMail(이메일 발송) |
| **실시간 통신** | Spring WebSocket + SockJS |
| **직렬화/JSON** | Jackson, Gson, json-simple |
| **빌드/환경** | Maven, Apache Tomcat, Log4j |

<br>

## 4. 아키텍처

Spring MVC의 **Controller – Service – DAO(Mapper)** 계층형 구조로 설계했습니다.

```
com.movie.plex
├── admin           # 관리자 기능
├── boards          # 공지사항 / FAQ / Q&A 게시판
├── coupon          # 쿠폰
├── files           # 파일 업로드·다운로드 공통 모듈
├── interceptors    # 로그인/관리자 권한 체크
├── like            # 콘텐츠·리뷰 좋아요
├── movieBooks      # 예매 및 결제(아임포트 연동)
├── movies          # 영화 정보 (TMDB API 연동)
├── nestcontents    # ReviewNest 콘텐츠(영화/TV)
├── pages           # 페이징 처리
├── review          # 리뷰 및 댓글
├── theater         # 상영관·좌석 관리
├── users           # 회원가입·로그인·카카오 로그인
└── websocket       # 실시간 채팅
```

- **View**: JSP + JSTL, 화면별 templates(header/footer/사이드바) 재사용
- **Data Access**: MyBatis Mapper XML로 SQL 관리, Oracle DB 연동
- **외부 API 연동**: `RestTemplate`으로 TMDB API 호출 후 자체 DB에 캐싱, 아임포트 SDK로 결제 검증
- **인증/인가**: 세션 기반 로그인, `HandlerInterceptor`로 관리자 페이지 접근 제어

<br>

## 5. ERD

![MoviePlex ERD](https://github.com/user-attachments/assets/229388df-9bfb-47b0-8122-f27cbe9a04bc)

<br>

## 6. 팀원 및 담당 역할

| [김채윤](https://github.com/erica-co) | [양은영](https://github.com/yangtori0407) | [이민열](https://github.com/mireu930) |
| :--------: | :--------: | :--------: |
| <img src="https://avatars.githubusercontent.com/u/181097382" height="120"/> | <img src="https://avatars.githubusercontent.com/u/114906941" height="120"/> | <img src="https://github.com/mireu79/ios-rock-paper-scissors/assets/125941932/b4a69222-b338-4a7f-984c-be5bd78dc1d8" height="120"/> |

> 담당 역할은 각자 이력서/포트폴리오에 맞춰 별도로 기재해주세요. (예: 예매/결제, ReviewNest, 실시간 채팅·관리자 페이지 등)

<br>

## 7. 트러블 슈팅

- (예시) TMDB API 페이지네이션 처리 시 총 페이지 수를 먼저 조회한 뒤 반복 호출하여 전체 목록을 수집하도록 개선
- (예시) 좌석 선택 동시성 이슈 방지를 위한 결제 검증 로직 설계
- 프로젝트 진행 중 겪은 이슈와 해결 과정을 이곳에 구체적으로 추가하면 포트폴리오 신뢰도가 높아집니다.

<br>

## 8. 실행 화면

| 카카오 로그인 | 영화 예매 | 리뷰네스트 |
| :--------: | :--------: | :--------: |
| <img src="https://github.com/user-attachments/assets/6b558858-9ffb-44a4-b8a0-f4a59ba1a4e3" height="200"/> | <img src="https://github.com/user-attachments/assets/ffabad95-de8c-4263-975a-9a3248c4a443" height="200"/> | <img src="https://github.com/user-attachments/assets/e43ba862-edea-4cec-a045-3b1f2653494d" height="200"/> |

<br>
