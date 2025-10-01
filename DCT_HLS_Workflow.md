# HLS 기반 DCT 개발 로드맵

## 배경

* Verilog로 512-point FFT 경험 있음
* C/HLS는 초급~중급 수준
* 포인트 크기는 8x8~16x16 정도로 설정
* 접근 방식: 1D DCT -> 2D (행·열 분리) 방식

## 개요 (핵심 아이디어)

* separable DCT (행별 1D DCT -> transpose -> 열별 1D DCT)
* 알고리즘 추천: Loeffler 8-point DCT, Arai 8-point DCT
* 데이터형: ap_fixed 기반 고정소수점 (예: ap_fixed<18,4>)
* HLS 스타일: 스트리밍/데이터플로우 + 파이프라인 + 배열 분할
* 인터페이스: 초기 simple memory interface, 이후 AXI4-Stream/AXI4-Lite/DMA 확장 가능

## 단계별 계획

### 1. 준비 (1주)

* HLS 툴 설치/환경: Xilinx Vitis HLS 또는 Vivado HLS
* C언어 HLS 문법 복습 (hls::stream, #pragma HLS, ap_fixed, 인터페이스 데코레이터)
* 출력물: hello world 함수 합성 결과 확인

### 2. 1D DCT 소프트웨어 모델 (C로, 1주)

* 목표: 정확한 수치 동작을 보이는 C 모델 작성 (부동소수점)
* 구현 방식: 선택한 알고리즘으로 1D N-point DCT 구현
* 테스트: 단일 펄스, 서인/사인 입력, 랜덤 벡터 등
* 검증: MATLAB/NumPy와 비교
* 출력물: C 모델 + 테스트 벡터, 비교 스크립트

### 3. 고정소수점 전환 & 정밀도 탐색 (1주)

* 목표: ap_fixed 변환, 비트폭 실험 (16,18,20비트 등)
* 검증: MSE/PSNR 비교
* 출력물: 실사용 품질 vs 리소스 트레이드오프 결정

### 4. HLS 구현 (기본 합성, 1~2주)

* 목표: 1D DCT HLS C/C++ 작성 및 합성 → RTL 생성
* 핵심 기법: #pragma HLS pipeline, array_partition, unroll, 인터페이스 지정
* 스트리밍: hls::stream<ap_fixed<...>> 사용
* 검증: C 시뮬, C/RTL Co-Sim
* 출력물: 합성 가능한 HLS 소스 + RTL 리포트

### 5. 2D DCT (행·열 처리) + 트랜스포즈 (1주)

* 목표: 1D DCT를 기반으로 2D DCT 구현 (8x8 블록 단위)
* 고려사항: transpose 구현 (BRAM buffer, ping-pong buffer)
* HLS 팁: #pragma HLS dataflow 사용, rowDCT->transpose->colDCT->write 파이프라인
* 출력물: 2D DCT HLS IP + 시뮬 결과

### 6. 최적화 (Throughput/Resource 튜닝, 1~2주)

* 목표: 목표 처리율 달성 (1 pixel/cycle 등)
* 기법: 루프 unroll, 배열 partition, 버퍼/BRAM 매핑, 정수/소수부 비트 조정
* 검증: 합성 리포트 (II, LAT, BRAM/LUT 사용량)

### 7. 시스템 통합 & 인터페이스 (AXI-Stream, DMA, 1~2주)

* 목표: AXI4-Stream/AXI4-Lite 인터페이스 래핑 → DMA 통해 FFmpeg 등과 연동
* 작업: AXI input/output 변경, 컨트롤 레지스터 설정, 데이터 왕복 테스트
* 출력물: AXI 연동 가능한 IP

### 8. 검증(엔드-투-엔드, 1주)

* 목표: 영상 스트림 → DCT → 인코더 전체 워크플로우 테스트
* 예: YCbCr 블록 → DCT(IP) → 양자화 → 파일 저장 → 비교

## 총정리(기간 요약)

* 기본 기능(1D → 2D, HLS 합성, 시뮬): 4–6주
* 최적화 + 시스템 통합 + 보드검증: 6–10주
* 참고: Verilog FFT 경험이 있어 HLS 학습 곡선 완만, C/HLS 익숙해지는 초기 1주 필요

## 기술/코드 팁

* 알고리즘: Loeffler 8-point DCT (버터플라이+상수곱 최소화)
* 데이터 타입: #include <ap_fixed.h>, ap_fixed<18,6> 실험
* HLS 라이브러리: hls::stream<T>, ap_int, ap_fixed
* 성능 지시자 예시:

```
#pragma HLS INTERFACE axis port=din
#pragma HLS INTERFACE axis port=dout
#pragma HLS INTERFACE s_axilite port=return bundle=CTRL
#pragma HLS DATAFLOW
```

* 버퍼(트랜스포즈): BRAM 기반 double-buffer, ping-pong buffer, ap_shift_reg 사용 고려
* 검증: cosim으로 RTL과 C 모델 일치 확인, PSNR/MSE 체크
* 리소스 목표: 8x8 DCT LUT 몇천, DSP 몇개, BRAM 몇개 수준 (합성 후 확인)

## 위험요소 및 대처

* 정밀도 문제: 고정소수점 비트폭 설계 실패 시 아티팩트 발생 → 여러 폭 실험
* 지연 증가: transpose/대기 버퍼로 latency 증가 → 데이터flow 재구성, BRAM 최적화
* HLS 숙련도: pragma 잘못 적용 시 합성 비효율 → 예제 참고, 리포트 확인

## 검증용 테스트아이디어

* 단일 펄스 이미지 → DCT 결과 예상 패턴 확인
* 서인(sinusoid) 입력 → 특정 빈도 성분 확인
* 실제 이미지 패치 → PSNR 비교

## 다음 단계 (옵션)

* 빠른 시작 템플릿: 1D DCT HLS C + 테스트벡터 제공
* 비트폭/정밀도 실험 스크립트: ap_fixed 조합 테스트 자동화
* Loeffler 8-point DCT C 구현 (부동 → 고정 변환 포함) 제공 가능
