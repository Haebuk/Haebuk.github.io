---
layout: single
title: "[Dask] Dask 튜토리얼"
categories: Python # 카테고리
tag: [Dask]
toc: true # 오른쪽에 table of contents 를 보여줌
author_profile: true # 게시글 들어갔을 때 좌측에 프로필 가리기
--- 
# 들어가며
현업에서 간단하게 로컬에서 데이터를 뽑아보려해도 수 GB는 훌쩍 넘어가는 경우가 다반사기 때문에, Pandas로는 한계가 있음을 느꼈습니다.

Dask를 사용하여 기초적인 병렬 계산, 데이터프레임 다루기, 간단한 신경망을 통해 학습하는 과정을 살펴보겠습니다.

https://www.youtube.com/watch?v=Alwgx_1qsj4를 참고했습니다.

예전에 촬영되어서 그대로 코드를 작성하면 작동하지 않는 코드가 여럿 있습니다. 2022년 1월 10일 기준으로 작동하도록 수정했습니다.

# Pre-required
dask와 함께 진행에는 영향이 없지만 아래에서 제공하는 시각화를 위해서는 graphviz 라이브러리를 설치해야합니다.

또한 Machine Learning 파트에서 Tensorflow를 사용합니다. M1 맥북에서 실행했기 때문에 출력문에 약간의 차이가 발생할 수 있습니다.

# Basic
첫번째로 dask가 제공하는 병렬 계산에 대해 살펴보도록 하겠습니다.

아래와 같이 함수가 작동할 때 마다 1초씩 대기하는 코드가 있습니다.

```python
from time import sleep

def inc(x):
    sleep(1)
    return x + 1

def add(x, y):
    sleep(1)
    return x + y
```

한번 실행시켜 보겠습니다. x와 y에 1과 2를 할당하고 x와 y를 더합니다.

```python
%%time

x = inc(1)
y = inc(2)
z = add(x, y)
```

    CPU times: user 451 µs, sys: 697 µs, total: 1.15 ms
    Wall time: 3.01 s

1초, 1초, 1초 3번을 대기 했기때문에 총 실행시간이 약 3초가 나왔음을 알 수 있습니다.

이를 Dask를 이용하여 기다리지 않고 계산하게 만들 수 있습니다.

```python
from dask import delayed
```

이를 위해서 Dask의 `delayed` 모듈을 임포트합니다. delayed 모듈은 병렬로 계산하고자 하는 것이 있을 때 매우 효과적입니다.

```python
@delayed
def delayed_inc(x):
    sleep(1)
    return x + 1

@delayed
def delayed_add(x, y):
    sleep(1)
    return x + y
```
위에서 정의했던 `inc`와 `add`의 함수와 동일합니다. 단지 함수 위에 `@delayed` 데코레이터를 붙여주기만 하면 끝입니다.

한 번 시간을 측정해보겠습니다.


```python
%%time
x = delayed(delayed_inc)(1)
y = delayed(delayed_inc)(2)
z = delayed(delayed_add)(x, y)
```
`delayed`메서드 안에 위에서 정의한 함수를 넣고 바깥에 함수 값을 할당합니다.

    CPU times: user 116 µs, sys: 20 µs, total: 136 µs
    Wall time: 130 µs

놀랍게도 1초도 안걸려 모든 계산이 끝났습니다. (정확한 결론은 아래를 참조해주세요.)

dask가 어떤 병렬 계산을 수행했는지 시각적으로 확인할 수 있습니다.

```python
z.visualize()
```

    
![png](/images/dask-tutorial_files/dask-tutorial_7_0.png)

위에서부터 차례로 코드를 실행하는 것이 아닌 병렬로 계산한다는 것을 알 수 있습니다.

--- 

dask의 메서드로 정의한 값을 알아보려면 평소와는 다른 방법을 써야하는데요, 아래와 같습니다.

`z`값 (2+3=5)가 나오길 기대했지만, 엉뚱한 값이 나옵니다.


```python
z
```




    Delayed('delayed_add-0e54f9e1-941d-49e9-903f-34e96b0dba54')

이는 실제 계산이 수행된 것이 아닌 어떤 메타데이터를 가르키고 있다고 볼 수 있습니다.

우리가 원하는 계산을 수행하려면 `compute()`를 사용해야 합니다.


```python
%%time
z.compute()
```

    CPU times: user 1.76 ms, sys: 1.56 ms, total: 3.32 ms
    Wall time: 2.01 s

    5

`z`의 값은 5가 나왔고, 실행 시간은 2초가 나왔습니다.
각 `inc(x)` `inc(y)`가 병렬로 수행되는 데 1초, `add(x+y)`에서 1초가 소요되었기 때문입니다.

# For loop
조금 더 오래걸리는 예제를 살펴보겠습니다.

파이썬의 for문은 악명이 자자한데요, 데이터 수를 무자비하게 늘리기보다는 앞에서 사용했던 함수를 사용해 시간을 늘려보겠습니다.

1부터 8까지 담겨져 있는 파이썬 리스트를 선언합니다.

```python
data = [1, 2, 3, 4, 5, 6, 7, 8]
```

리스트에서 값을 뽑아 `inc`함수에 삽입하고 결과를 빈 리스트에 담아 최종 합을 산출하는 코드입니다.
```python
%%time

results = []
for x in data:
    y = inc(x)
    results.append(y)

total = sum(results)
```

    CPU times: user 1.19 ms, sys: 1.2 ms, total: 2.38 ms
    Wall time: 8.03 s

`inc`가 총 8번 호출됐기 때문에 실행시간이 8초가 나왔습니다.

이를 dask의 delayed 메서드로 병렬화시켜보겠습니다. 과연 모든 `inc`가 병렬로 계산되어 1초 남짓한 시간이 걸릴까요?

```python
%%time

results = []

for x in data:
    y = delayed(delayed_inc)(x)
    results.append(y)

total = delayed(sum)(results)

total.compute()
```

    CPU times: user 3.28 ms, sys: 2.15 ms, total: 5.43 ms
    Wall time: 1.01 s

    44

1부터 8까지 모두 더한 44가 결과값으로 나왔고, 실행시간은 예상한대로 1초가 나왔습니다. 모든 `inc` 함수가 병렬로 수행됐음이 분명합니다.

검증을 위해 시각화해보겠습니다.


```python
total.visualize()
```


![png](/images/dask-tutorial_files/dask-tutorial_13_0.png)
    

처음에 이 그래프를 보고 감탄했던 기억이(...) 아름답게 병렬 계산을 하는 것을 알 수 있습니다.

# DataFrame
dask하면 생각나는 것이 바로 대용량 dataframe입니다. dask를 이용해 빠르게 로드하고 집계하는 방법에 대해 살펴보겠습니다.

먼저 데이터는 약 200MB의 데이터로 뉴욕 공항의 항공기 이착륙 관련 데이터입니다.

아래와 같이 데이터를 다운로드 하고 로드합니다.

```python
import urllib

print("- Downloading NYC Flights dataset... ", end='', flush=True)
url = "https://storage.googleapis.com/dask-tutorial-data/nycflights.tar.gz"
filename, headers = urllib.request.urlretrieve(url, 'nycflights.tar.gz')
```

    - Downloading NYC Flights dataset... 


```python
import tarfile

with tarfile.open(filename, mode='r:gz') as flights:
    flights.extractall('data/')
```
여기까지 했다면 data폴더 안에 10개의 csv파일이 생성된 것을 확인할 수 있습니다.

pandas를 사용할 경우 이를 반복문을 통해 `pd.concat`으로 순차적으로 데이터프레임을 합치는 방법으로 접근하는데요, dask는 아래와 같이 분할된 파일을 한번에 로드할 수 있는 기능을 제공합니다.

```python
import os
import dask.dataframe as dd

df = dd.read_csv(os.path.join('data', 'nycflights', '*.csv'), parse_dates={'Date': [0, 1, 2]})

df
```




<div><strong>Dask DataFrame Structure:</strong></div>
<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>DayOfWeek</th>
      <th>DepTime</th>
      <th>CRSDepTime</th>
      <th>ArrTime</th>
      <th>CRSArrTime</th>
      <th>UniqueCarrier</th>
      <th>FlightNum</th>
      <th>TailNum</th>
      <th>ActualElapsedTime</th>
      <th>CRSElapsedTime</th>
      <th>AirTime</th>
      <th>ArrDelay</th>
      <th>DepDelay</th>
      <th>Origin</th>
      <th>Dest</th>
      <th>Distance</th>
      <th>TaxiIn</th>
      <th>TaxiOut</th>
      <th>Cancelled</th>
      <th>Diverted</th>
    </tr>
    <tr>
      <th>npartitions=10</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th></th>
      <td>datetime64[ns]</td>
      <td>int64</td>
      <td>float64</td>
      <td>int64</td>
      <td>float64</td>
      <td>int64</td>
      <td>object</td>
      <td>int64</td>
      <td>float64</td>
      <td>float64</td>
      <td>int64</td>
      <td>float64</td>
      <td>float64</td>
      <td>float64</td>
      <td>object</td>
      <td>object</td>
      <td>float64</td>
      <td>float64</td>
      <td>float64</td>
      <td>int64</td>
      <td>int64</td>
    </tr>
    <tr>
      <th></th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th></th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th></th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
  </tbody>
</table>
</div>
<div>Dask Name: read-csv, 10 tasks</div>
---
매우 빠르게 실행이 되는데요,  `npartition=10`을 통해 10개의 파티션으로 이루어져있다는 것이 특징입니다.

pandas와는 다르게 모든 값이 감춰져있습니다. 이를 살펴보는 방법은 조금 후에 살펴보도록 하고, dask 데이터프레임 로드시 주의해야할 점에 대해 먼저 설명하겠습니다.


```python
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>DayOfWeek</th>
      <th>DepTime</th>
      <th>CRSDepTime</th>
      <th>ArrTime</th>
      <th>CRSArrTime</th>
      <th>UniqueCarrier</th>
      <th>FlightNum</th>
      <th>TailNum</th>
      <th>ActualElapsedTime</th>
      <th>...</th>
      <th>AirTime</th>
      <th>ArrDelay</th>
      <th>DepDelay</th>
      <th>Origin</th>
      <th>Dest</th>
      <th>Distance</th>
      <th>TaxiIn</th>
      <th>TaxiOut</th>
      <th>Cancelled</th>
      <th>Diverted</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1990-01-01</td>
      <td>1</td>
      <td>1621.0</td>
      <td>1540</td>
      <td>1747.0</td>
      <td>1701</td>
      <td>US</td>
      <td>33</td>
      <td>NaN</td>
      <td>86.0</td>
      <td>...</td>
      <td>NaN</td>
      <td>46.0</td>
      <td>41.0</td>
      <td>EWR</td>
      <td>PIT</td>
      <td>319.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1990-01-02</td>
      <td>2</td>
      <td>1547.0</td>
      <td>1540</td>
      <td>1700.0</td>
      <td>1701</td>
      <td>US</td>
      <td>33</td>
      <td>NaN</td>
      <td>73.0</td>
      <td>...</td>
      <td>NaN</td>
      <td>-1.0</td>
      <td>7.0</td>
      <td>EWR</td>
      <td>PIT</td>
      <td>319.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1990-01-03</td>
      <td>3</td>
      <td>1546.0</td>
      <td>1540</td>
      <td>1710.0</td>
      <td>1701</td>
      <td>US</td>
      <td>33</td>
      <td>NaN</td>
      <td>84.0</td>
      <td>...</td>
      <td>NaN</td>
      <td>9.0</td>
      <td>6.0</td>
      <td>EWR</td>
      <td>PIT</td>
      <td>319.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1990-01-04</td>
      <td>4</td>
      <td>1542.0</td>
      <td>1540</td>
      <td>1710.0</td>
      <td>1701</td>
      <td>US</td>
      <td>33</td>
      <td>NaN</td>
      <td>88.0</td>
      <td>...</td>
      <td>NaN</td>
      <td>9.0</td>
      <td>2.0</td>
      <td>EWR</td>
      <td>PIT</td>
      <td>319.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1990-01-05</td>
      <td>5</td>
      <td>1549.0</td>
      <td>1540</td>
      <td>1706.0</td>
      <td>1701</td>
      <td>US</td>
      <td>33</td>
      <td>NaN</td>
      <td>77.0</td>
      <td>...</td>
      <td>NaN</td>
      <td>5.0</td>
      <td>9.0</td>
      <td>EWR</td>
      <td>PIT</td>
      <td>319.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>0</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 21 columns</p>
</div>

`head()`메서드는 pandas와 동일한 기능을 제공합니다. `tail()`도 한 번 살펴볼까요?


```python
df.tail()
```


    ---------------------------------------------------------------------------

    ValueError                                Traceback (most recent call last)

    /var/folders/xm/8mvqw44j1md_q70lrkm9_wh00000gn/T/ipykernel_47466/281403043.py in <module>
    ----> 1 df.tail()
    

    /opt/homebrew/Caskroom/miniforge/base/envs/tensorflow/lib/python3.9/site-packages/dask/dataframe/core.py in tail(self, n, compute)
       1143 
       1144         if compute:
    -> 1145             result = result.compute()
       1146         return result
       1147 

    ... 중략 ...

    ValueError: Mismatched dtypes found in `pd.read_csv`/`pd.read_table`.
    
    +----------------+---------+----------+
    | Column         | Found   | Expected |
    +----------------+---------+----------+
    | CRSElapsedTime | float64 | int64    |
    | TailNum        | object  | float64  |
    +----------------+---------+----------+
    
    The following columns also raised exceptions on conversion:
    
    - TailNum
      ValueError("could not convert string to float: 'N54711'")
    
    Usually this is due to dask's dtype inference failing, and
    *may* be fixed by specifying dtypes manually by adding:
    
    dtype={'CRSElapsedTime': 'float64',
           'TailNum': 'object'}
    
    to the call to `read_csv`/`read_table`.

`tail()`은 `head()`와 다르게 오류가 발생합니다. 이는 dask가 dataframe을 생성할 때 데이터타입을 데이터의 초반 행을 통해 추론하기 때문입니다.

오류문을 살펴보면 `CRSElapsedTime`은 int64를 기대했는데 float64였고, `TailNum`은 float64를 기대했는데 object가 나타났다고 합니다.

이를 해결하기 위해서는 dask dataframe을 정의할 때 데이터타입을 명시해줘야 합니다.

```python
df = dd.read_csv(
    os.path.join('data', 'nycflights', '*.csv'),
    parse_dates={'Date': [0, 1, 2]},
    dtype={'TailNum': str,
    'CRSElapsedTime': float,
    'Cancelled': bool}
)
```
`CRSElapsedTime`과 `TailNum`의 데이터 타입을 명시하고, 이따가 사용할 `Cancelled`열도 미리 데이터 타입을 선언해주겠습니다.

다시 한 번 마지막 값을 살펴보겠습니다.

```python
df.tail()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Date</th>
      <th>DayOfWeek</th>
      <th>DepTime</th>
      <th>CRSDepTime</th>
      <th>ArrTime</th>
      <th>CRSArrTime</th>
      <th>UniqueCarrier</th>
      <th>FlightNum</th>
      <th>TailNum</th>
      <th>ActualElapsedTime</th>
      <th>...</th>
      <th>AirTime</th>
      <th>ArrDelay</th>
      <th>DepDelay</th>
      <th>Origin</th>
      <th>Dest</th>
      <th>Distance</th>
      <th>TaxiIn</th>
      <th>TaxiOut</th>
      <th>Cancelled</th>
      <th>Diverted</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>269176</th>
      <td>1999-12-27</td>
      <td>1</td>
      <td>1645.0</td>
      <td>1645</td>
      <td>1830.0</td>
      <td>1901</td>
      <td>UA</td>
      <td>1753</td>
      <td>N516UA</td>
      <td>225.0</td>
      <td>...</td>
      <td>205.0</td>
      <td>-31.0</td>
      <td>0.0</td>
      <td>LGA</td>
      <td>DEN</td>
      <td>1619.0</td>
      <td>7.0</td>
      <td>13.0</td>
      <td>False</td>
      <td>0</td>
    </tr>
    <tr>
      <th>269177</th>
      <td>1999-12-28</td>
      <td>2</td>
      <td>1726.0</td>
      <td>1645</td>
      <td>1928.0</td>
      <td>1901</td>
      <td>UA</td>
      <td>1753</td>
      <td>N504UA</td>
      <td>242.0</td>
      <td>...</td>
      <td>214.0</td>
      <td>27.0</td>
      <td>41.0</td>
      <td>LGA</td>
      <td>DEN</td>
      <td>1619.0</td>
      <td>5.0</td>
      <td>23.0</td>
      <td>False</td>
      <td>0</td>
    </tr>
    <tr>
      <th>269178</th>
      <td>1999-12-29</td>
      <td>3</td>
      <td>1646.0</td>
      <td>1645</td>
      <td>1846.0</td>
      <td>1901</td>
      <td>UA</td>
      <td>1753</td>
      <td>N592UA</td>
      <td>240.0</td>
      <td>...</td>
      <td>220.0</td>
      <td>-15.0</td>
      <td>1.0</td>
      <td>LGA</td>
      <td>DEN</td>
      <td>1619.0</td>
      <td>5.0</td>
      <td>15.0</td>
      <td>False</td>
      <td>0</td>
    </tr>
    <tr>
      <th>269179</th>
      <td>1999-12-30</td>
      <td>4</td>
      <td>1651.0</td>
      <td>1645</td>
      <td>1908.0</td>
      <td>1901</td>
      <td>UA</td>
      <td>1753</td>
      <td>N575UA</td>
      <td>257.0</td>
      <td>...</td>
      <td>233.0</td>
      <td>7.0</td>
      <td>6.0</td>
      <td>LGA</td>
      <td>DEN</td>
      <td>1619.0</td>
      <td>5.0</td>
      <td>19.0</td>
      <td>False</td>
      <td>0</td>
    </tr>
    <tr>
      <th>269180</th>
      <td>1999-12-31</td>
      <td>5</td>
      <td>1642.0</td>
      <td>1645</td>
      <td>1851.0</td>
      <td>1901</td>
      <td>UA</td>
      <td>1753</td>
      <td>N539UA</td>
      <td>249.0</td>
      <td>...</td>
      <td>232.0</td>
      <td>-10.0</td>
      <td>-3.0</td>
      <td>LGA</td>
      <td>DEN</td>
      <td>1619.0</td>
      <td>6.0</td>
      <td>11.0</td>
      <td>False</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 21 columns</p>
</div>

정상적으로 값이 출력됨을 알 수 있습니다.

---

이번에는 간단한 집계함수를 사용해보겠습니다.

```python
%time df.DepDelay.max().compute()
```

    CPU times: user 3.18 s, sys: 526 ms, total: 3.71 s
    Wall time: 1.61 s

    1435.0

매우 빠르게 최대값을 산출해냄을 알 수 있습니다. dask는 이를 어떻게 계산했을까요?

시각적으로 살펴보겠습니다.


```python
df.DepDelay.max().visualize(rankdir='LR', size='12, 12!')
```




    
![png](/images/dask-tutorial_files/dask-tutorial_23_0.png)
    
각각의 파티션(총 10개)에서 최대값 후보를 선정한다음에 최종 최대값을 선출해냄을 알 수 있습니다.

단순하게 생각하면 pandas 집계보다 10배 빠르다고 볼 수도 있겠습니다.


# ML with Dask

마지막으로 간단한 신경망을 통해 학습하는 방법을 살펴보고 마치겠습니다.

먼저 학습에 사용할 데이터를 정의하고 정보를 확인합니다.

```python
df_train = df[['CRSDepTime', 'CRSArrTime', 'Cancelled']]
```


```python
df_train.iloc[:, :-1].compute().values
```




    array([[1540, 1701],
           [1540, 1701],
           [1540, 1701],
           ...,
           [1645, 1901],
           [1645, 1901],
           [1645, 1901]])




```python
df_train.iloc[:, -1].compute().values
```




    array([False, False, False, ..., False, False, False])

```python
df_train.shape
```

    (Delayed('int-94ab9ac8-9432-4a95-b40f-abdaca09c41e'), 3)

0번째 값은 dask delayed객체로 나오고, 1번째 값은 총 열 개수인 3이 나오는 것이 특징입니다.


```python
df_train.isnull().sum().compute()
```




    CRSDepTime    0
    CRSArrTime    0
    Cancelled     0
    dtype: int64

결측치는 없습니다. 아주 간단한 신경망을 정의하고 학습시켜보겠습니다.

```python
import tensorflow as tf
from keras.models import Sequential
from keras.layers import Dense

model = Sequential()
model.add(Dense(20, input_dim=df_train.shape[1]-1, activation='relu'))
model.add(Dense(1, activation='sigmoid'))

model.compile(loss='binary_crossentropy', optimizer='sgd')
```


`from_tensor_slices`를 사용해 데이터프레임을 변환합니다.

```python
dataset = tf.data.Dataset.from_tensor_slices(
    (df_train.iloc[:, :-1].compute().values, df_train.iloc[:, -1].compute().values)
).batch(512)
```


```python
model.fit(dataset, epochs=5)
```

    Epoch 1/5
    10203/10203 [==============================] - 38s 4ms/step - loss: 239.6750
    Epoch 2/5
    10203/10203 [==============================] - 38s 4ms/step - loss: 0.1011
    Epoch 3/5
    10203/10203 [==============================] - 38s 4ms/step - loss: 0.1006
    Epoch 4/5
    10203/10203 [==============================] - 39s 4ms/step - loss: 0.1006
    Epoch 5/5
    10203/10203 [==============================] - 39s 4ms/step - loss: 0.1006

    <keras.callbacks.History at 0x298c5f040>

여기서는 학습을 할 수 있다에 초점을 맞췄기 때문에, 성능 검증은 다루지 않습니다.

# 마치며
대용량 데이터에 적합한 라이브러리인 dask에 대해 기초를 다뤄봤습니다. 심화된 기능은 공식 홈페이지에 상세히 나와있습니다.

dask의 기능을 좀 더 숙지한다면 매우 많은 부분에서 pandas를 대체할 수 있을것이라 기대합니다.

더 소개할만한 기능을 수집해서 다음 포스팅에 공유하도록 하겠습니다. 감사합니다.