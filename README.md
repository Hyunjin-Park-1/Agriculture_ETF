# LightGBM 기반 농산물 선물 ETF 가격 등락 예측 모델

## 1. 프로젝트 설명
---

1) 장기 지속중인 코로나 팬데믹으로 인해 각국의 통화증발과 경기부양책 사용이 장기화

2) 인플레이션 압박 심화와 더불어 Fed의 테이퍼링 정책과 기준금리 인상으로 양방향 물가변동 리스크를 헤지하기 위한 전략의 필요성 대두

3) 물가 변동 리스크 헤지를 위한 대체투자상품 중 농산물 선물 상품에 주목

4) 시카고 상품거래소(CBOT) 내의 대두, 밀, 옥수수 등의 선물상품으로 구성된 농산물선물지수인 S&P GSCI Agriculture Enhanced Select Index (ER)를 추종하는 지수추종펀드(ETF)에 주목

5) 환헤지 효과를 누리기 위해 GSCI 지수를 추종하는 국내 ETF를 선택

6) Random Forest 기반의 LightGBM 모델을 활용하여 익영업일의 가격 등락 예측, 예측 결과를 기반으로 간단한 백테스팅 진행

7) 22년도 제12회 DB금융경제공모전 가작 수상, 이후 코드 수정 및 정리, 다양한 시도를 첨부함

## 2. 사용 Data 및 Feature
---

1) TIGER ETF 및 KODEX ETF : 현 가격 및 lag값 기반으로 등락을 예측하기 위해 사용

2) S&P500, S&P500 선물 : 주식시장 자체의 강세/약세를 반영하기 위해 사용

3) GDP growth : 경기의 호불황에 대한 예측과 실제 간의 괴리가 선물 가격에 즉각적으로 반영되는 것을 고려하기 위해 사용

4) CBOT 및 CME 곡물선물 : 기초자산의 가격흐름을 추적하기 위해 사용

5) 원유/에탄올/국채선물 : 곡물선물과 유의미한 관계를 가지고 있는 것으로 알려진 원유/대체에너지/실질금리(10년만기 미국 국채로 대체) 가격 정보를 반영하기 위해 사용

6) 코덱스 및 타이거의 철강/금속/선물환 ETF : 외국의 지수를 국내에서 추종할 때, 환리스크 및 국제정세 리스크로 인해 조정이 일어나는 과정을 반영하기 위해 사용

7) GSCI : 기초지수 가격흐름 정보를 반영하기 위해 사용

8) 미국 농산물선물 ETF : 농업 활동에 전반적인 영향을 미치는 기후, 자연재해, 비료 가격 등락 등 다양한 외부 정보를 반영하기 위해 사용

**Featrues Lists**
![features1](https://user-images.githubusercontent.com/90302595/179939952-ba0842b6-1fd7-47a5-be48-b7021d9a2b75.png)
![features2](https://user-images.githubusercontent.com/90302595/179940179-c6bfc0e1-3588-4327-ac3c-136c420b880a.png)
![features3](https://user-images.githubusercontent.com/90302595/179940190-e780f5c4-7c1b-4e05-a6b7-9518a08b51cc.png)
![features4](https://user-images.githubusercontent.com/90302595/179940198-9077e98a-6014-4dc3-8602-4c3419f6a483.png)

cf) 추후 피처 엔지니어링 과정에서 이동평균 및 레그값을 추가함

cf) 데이터 출처 : Yahoo Finance, Naver 금융(pandas-datareader 패키지 이용) 및 Fed, Investing.com

## 3. 데이터 전처리
---

1) 결측치 처리

- 전반적으로 결측치는 많지 않음 (거래량 : 각 모델의 데이터프레임에서 컬럼별 2~3개, 에탄올 선물만 거래량 결측치가 368개)
- 거래량 결측치는 해당일의 거래량이 매우 적거나 거래가 실시되지 않은 것으로 보고 0으로 대체
- 종가 결측치는 bias를 최소화하기 위해 이전 영업일의 종가로 처리 (ffill)
- gdp 성장률 발표 데이터는 발표일과 발표일 사이 결측치를, 직전 발표일의 정보로 채움 (ffill)

2) 데이터 병합

- 타이거/코덱스 ETF 가격 정보가 존재하는 거래일을 기준으로 각 데이터 병합
- 두 모델의 데이터프레임에서 각각 데이터가 누락된 270일 / 83일의 데이터 drop

3) Severe Outlier 처리

*유가선물 종가/유가선물 거래량/타이거 금속선물ETF 거래량/타이거 원유선물 거래량/KOSEF 달러선물 거래량* 

- 제일 극단값이 심했던 다섯 개의 컬럼에 대해, clip method를 이용하여 Winsorizing 실시
```
df1['tiger_oil_volume']=df1['tiger_oil_volume'].clip(upper=up_o_vol.max())
```
4) 20일 이동평균 / lag값 추가
- 데이터 중 각종 종가 컬럼에 대해 과거 가격의 강세/약세 추세를 반영하기 위해 20일 이동평균을 계산해서 컬럼에 추가 (eg. kodex_close_mean)
- TIGER ETF 종가 및 KODEX ETF 종가 컬럼에 대해 최근 가격 추세를 반영하기 위해 1, 2, 3 영업일 전의 종가 (각각 lag_1, lag_2, lag_3) 컬럼을 추가

5) 정규화
- 상대적으로 길이가 긴 TIGER 자료는 Standard 정규화를, 길이가 짧은 KODEX 자료는 Robust 정규화를 통해 정규화

6) Feature Importance 하위 5개 컬럼 drop

- TIGER 모델 : *sp500_adjclose / actual_growth / DBA_adjclose_mean / crude_oil_adjclose_mean / sp500_adjclose_mean*

- KODEX 모델 : *CORN_adjclose_mean / sp500_adjclose_mean / tbond_fut_adjclose / prev_growth / actual_growth*

## 4. 모델 학습 및 HPO 실시
---
- 각 모델의 F1 Score 값 :</br>
a) TIGER 모델 : 0.7031
b) KODEX 모델 : 0.6560

## 5. 결과 해석
---
### TIGER ETF 모델
1) TIGER 예측 모델의 경우 가장 중요한 feature는 시카고 선물시장의 옥수수 선물의 종가인 corn_close임
    - TIGER 농산물선물 ETF를 구성하는 종목 중 옥수수 선물이 가장 큰 비중 (30% 상회)을 차지하기 때문
    - 옥수수 선물의 이동평균값 또한 중요한 signal 로서 작용하고 있음

2) 전 영업일 뿐 아니라 전전영업일(lag-2), 그 전영업일(lag-3) 역시 유의미한 feature 
importance 점수를 획득하였지만  FI 순위가 높지는 않음

3) 전반적으로 등락 예측을 할 때 ETF의 종가 등 절대적인 가격보다는 거래량이 예측에 더 중요한 정보임

4) 모델이 거의 모든 feature를 활용하여 등락을 예측하도록 학습됨

### KODEX ETF 모델
1) 하이퍼파라미터 튜닝 결과 모델이 많은 feature를 아예 사용하지 않음
(FI가 0인 feature가 많음)

    - TIGER 모델에 반해, 모델이 모든 컬럼의 정보를 활용하여 학습을 진행할 만큼 데이터가 충분하지 않기 때문인 것으로 보임

2) TIGER와 마찬가지로 corn_close의 feature importance가 가장 높음.

3) 특이사항으로는, ETF 자체의 종가의 FI가 그리 높진 않던 TIGER에 반해, KODEX는 기초지수인 gsci와 KODEX 종가의 FI가 상당히 높음.
    - 위 1번과 같은 맥락에서, 모델이 모든 데이터를 충분히 활용하여 학습하기보다는 당일의 가격과 추종지수(gsci) 값만을 사용하여 익영업일의 등락을 추론하도록 학습된 것으로 보임
4) KODEX ETF는 여유자금을 단기 우량채 및 예금으로 운용한다는 특징을 가지고 있음. 그러나, 실제 모델에서는 tbond_fut_adjclose 컬럼의 feature 점수가 매우 낮게 나타남

    - 중요한 feature임에도 불구하고 데이터의 길이가 충분하지 않으면 모델이 구조로부터 충분한 정보를 캐낼 수 없다는 것을 확인할 수 있음
    - 비교적 최근에 상장된 상품, 혹은 거래량이 부진한 상품에 대해서는 Random Forest 기반의 prediction의 적용이 어려울 가능성이 높음

## 6. 예측값을 활용한 간단한 백테스팅
---
1) LightGBM과 데이터의 Test set을 이용하여 익영업일의 각 ETF의 가격 등락 여부를 예측

2) 가격 등락 여부를 예측한 결과 가격 상승이 예상된 경우 매수, 가격 하락이 예상된 경우 매도 (최대 포지션: 1주, 공매도 x 가정)

3) 슬리피지, 수수료 등을 고려하여 매 거래 시 거래비용 0.015%가 발생한다고 단순화 (삼성증권 ETF 거래수수료 참조)

4) 과정
- pct_change() 함수로 일일 수익률을 구하고, 이를 누적곱하여 단순 보유 수익률인 `return` 컬럼을 계산
- pct_change() 결과 중 예측값이 1인 수익률만 누적곱하여 모델에 따른 수익률인 `strategy` 컬럼을 계산 (공매도 불가 제약)
- 전체 기간에 대한 누적 수익률 값과, 기간에 따른 누적수익률 그래프를 그려 비교

**TIGER ETF의 백테스팅 결과**
- 단순 보유 : 67.1533 %
- 모델 기반 : 66.0898 %

**KODEX ETF의 백테스팅 결과**
- 단순 보유 : 20.6234 %
- 모델 기반 : 34.7657 %

5) 결과 해석</br>
    a) TIGER ETF의 경우 2021년 1월까지는 모델에 기반한 전략의 수익률이 단순 보유 전략의 수익률을 상회

    b) 그러나 2021년 1월부터 단순 보유전략을 하회하는 수익률을 보이다가, 투자기간 말기에 수렴하는 양상을 보임

    c) KODEX ETF의 경우 2021년 3월 이후 모델에 기반한 전략이 단순 보유 전략을 크게 상회하면서, 투자기간 말기에는 모델 기반 전략이 단순 보유 전략의 1.5배를 상회하는 수익률을 보임

    d) f1 score 자체는 (TIGER: 0.703) vs (KODEX: 0.658)로 TIGER 모델이 더 높지만, 단순 long-only 전략에 적용했을 때에는 오히려 KODEX 모델의 수익률이 더 우월함. 

    => 단순 등락 예측을 단일 signal로 사용하는 것보다는, 여러 Factor 중 하나로 사용하는 것이 좋아 보인다.

## 7. 시사점
---
1) 가격 및 거래량 정보가 금융 시계열 자료임에도 불구하고, 트리 기반의 분류 모델의 적용이 유의미한 예측을 산출할 수 있음을 확인함
2) 트리 모델을 사용하면 기존 연구에서 밝혀진 데이터 간 종속 구조를 쉽게 활용해 포트폴리오 구성 전략을 보강할 수 있을 것으로 보임
3) 상장일이 비교적 최근인 신생 상품보다는 거래 정보가 충분히 축적된 상품에 대해 트리 모델을 적용하면 상품의 구조 정보에 대한 자세한 분석이 가능함
4) 새로운 intuition을 찾아낼 수 있는 도구로 활용될 수 있음</br>
Eg)</br>
    - FI 분석을 통해 절대적인 가격보다는 거래량 정보가 농산물선물 가격 등락 예측에 중요하다는 사실을 발견 
    - 외부 충격에 큰 영향을 받는 농산물 선물의 경우, 전통적인 Factor인 이동평균이나 Moment, RSI, MACD보다는 거래량으로부터 외부 충격에 대한 선제적인 움직임을 감지할 수 있을 것으로 보임
    - 여유자금 운용 방법 자체는 ETF의 가격등락에 큰 영향을 미치지 못함
    - 포트폴리오에 포함되어 있지 않은 에탄올 선물 거래량 컬럼의 FI가 상위에 위치한 것으로 보아, 대체에너지 수요가 농산물 선물과 유의미한 상관관계가 있을 것으로 보임
5) 한계점 및 추가 고려사항
- 기후 데이터는 농산물선물 가격 결정에 있어 핵심적인 요소지만, 본 모델에서는 다른 농산물선물 ETF 정보로 이를 대체함 (CORN, DBA 등 해외 농산물 ETF)
- 표준화된 기후 예보 및 실제 기후 데이터 사용시 모델 성능이 향상될 것으로 기대
- 본 모델에서는 표준정규분포 정규화와 Robust 정규화를 통해 종가 데이터를 Scaling했지만, 시계열 데이터의 특성을 고려한 정규화 기법을 적용하면 모델 성능이 향상될 것으로 기대
- 백테스팅을 수행하기 위해 단순 일일 등락을 기준으로 한 거래를 상정함. 그러나 ETF가 passive한 자산운용을 위한 투자대상임을 고려해볼 때, 좀 더 보수적인 전략을 설계해볼 것이 권장됨</br>
    eg) 모델을 확장하여 n영업일 간의 예측치를 얻은 후, 연속 k일 간 가격하락 / 가격상승이 예측될 때에만 매도 / 매수하는 전략
- 투자 기간 내의 다양한 외부 충격으로 인해 가격이 비정상적으로 변동하는 것을 제어했으나, 단순 이동평균이 아닌 shock 기반으로 변동성 제어 기법을 도입해볼 여지가 있음
