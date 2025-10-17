# Xarray 학습 정리

## Xarray 개요
Xarray는 두 가지 주요 데이터 구조를 제공한다.
1. **DataArray** (기본 구성 요소)
2. **Dataset** (여러 데이터 변수를 모은 것)

> Series, DataArray, Dataset 객체는 주로 pandas, NetCDF, Zarr 같은 다른 라이브러리로부터 변환하여 생성됩니다.

### Pandas Series 예제
```python
series = pd.Series(np.ones((10,)), index=list("abcdefghij"))
# 출력:
# a    1.0
# b    1.0
# c    1.0
# ...
# j    1.0
# dtype: float64
```

---

## 1. DataArray

### 주요 구성 요소

#### 1.1 `.data` - 원시 수치 데이터
실제 배열 데이터가 저장되는 곳

#### 1.2 `.dims` - 명명된 차원
```python
da.dims 
# 출력: ('time', 'lat', 'lon')
```

#### 1.3 `.coords` - 좌표
각 차원을 따라 각 점을 라벨링하는 좌표
```python
da.coords
# 출력:
# Coordinates:
#   * lat      (lat) float32 100B 75.0 72.5 70.0 67.5 65.0 ... 22.5 20.0 17.5 15.0
#   * lon      (lon) float32 212B 200.0 202.5 205.0 207.5 ... 325.0 327.5 330.0
#   * time     (time) datetime64[ns] 23kB 2013-01-01 ... 2014-12-31T18:00:00
```

#### 1.4 `.attrs` - 메타데이터 딕셔너리
```python
{
    'long_name': '4xDaily Air temperature at sigma level 995',
    'units': 'degK',
    'precision': np.int16(2),
    'GRIB_id': np.int16(11),
    'GRIB_name': 'TMP',
    'var_desc': 'Air temperature',
    'dataset': 'NMC Reanalysis',
    'level_desc': 'Surface',
    'statistic': 'Individual Obs',
    'parent_stat': 'Other',
    'actual_range': array([185.16, 322.1], dtype=float32)
}
```

### DataArray 변환 예제

#### Pandas Series → Xarray DataArray
```python
arr = series.to_xarray()
# <xarray.DataArray (index: 10)> Size: 80B
# array([1., 1., 1., 1., 1., 1., 1., 1., 1., 1.])
# Coordinates:
#   * index    (index) object 80B 'a' 'b' 'c' 'd' 'e' 'f' 'g' 'h' 'i' 'j'
```

#### Xarray DataArray → Pandas Series
```python
arr.to_pandas()
# 또는
da.to_series()  # DataArray를 pandas Series로 변환
```

#### Xarray DataArray → DataFrame
```python
da.to_dataframe()
#                         air
# time        lat   lon  
# 2013-01-01  75.0  200.0  241.20
#                   202.5  242.50
# ...
```

### DataArray 생성 예제

#### 기본 생성
```python
xr.DataArray(array)
```

#### 차원과 좌표를 포함한 생성
```python
lon_values = np.arange(200, 331, 2.5)
lon_da = xr.DataArray(lon_values, dims='lon')

da = xr.DataArray(
    array, 
    dims=("times", "lat", "lon"), 
    coords={"lon": lon_da}, 
    attrs={"attribute": "something"}
)
```

### Non-dimension Coordinates
기존 차원을 따라 좌표 변수를 추가로 부착하고 싶을 때 사용
```python
# 'itime'은 차원 이름 'time'과 다른 이름을 가진 좌표
da.coords["itime"] = ("time", np.arange(2920), {"name": "value"})
```

### 종합 예제
```python
xr.DataArray(
    # 1. 데이터: 180x360 크기의 0~400 사이 랜덤 숫자
    rng.random((180, 360)) * 400,
    
    # 2. 차원 이름: 각 축에 'latitude', 'longitude' 이름 붙이기
    dims=("latitude", "longitude"),
    
    # 3. 좌표: 각 차원의 축에 실제 좌표 값과 정보(속성) 할당
    coords={
        "latitude": ("latitude", np.arange(-90, 90, 1), {"type": "geodetic"}),
        "longitude": ("longitude", np.arange(-180, 180, 1), {"prime_meridian": "greenwich"})
    },
    
    # 4. 데이터 전체의 속성: 이 데이터가 'ellipsoid' 타입임을 명시
    attrs={"type": "ellipsoid"},
    
    # 5. 데이터 이름: 이 DataArray의 변수명을 'height'로 지정
    name="height"
)
```

---

## 2. Dataset

### 주요 구성 요소
- **data_vars**: 이름과 값을 매핑하는 dict-like 객체

### Dataset 생성 예제

#### 기본 생성
```python
ds = xr.Dataset({"air": da, "air2": da2})

# 새로운 변수 추가
ds["air3"] = da
```

#### 좌표와 속성을 포함한 생성
```python
xr.Dataset(
    {"air": da, "air2": da2},
    coords={
        "time": pd.date_range("2013-01-01", "2014-12-31 18:00", freq="6H")
    },
    attrs={"key0": "value0"}
)
```

### Dataset 구성 요소
- **DataArray**: 객체 또는 튜플로 정의
- **coords**: 좌표 정보
- **attrs**: 속성 정보

---

## 3. NetCDF와 Zarr 파일 다루기

### 데이터 저장 및 불러오기

#### Dataset 생성 및 NetCDF로 저장
```python
datadir2 = pathlib.Path('../tutorial/data')

# Dataset 1 생성
ds1 = xr.Dataset(
    data_vars={
        "a": (("x", "y"), np.random.randn(4, 2)),
        "b": (("z", "x"), np.random.randn(6, 4))
    },
    coords={
        "x": np.arange(4),
        "y": np.arange(-2, 0),
        "z": np.arange(-3, 3)
    }
)
# NetCDF 파일로 저장
ds1.to_netcdf(datadir2/"ds1.nc")

# Dataset 2 생성
ds2 = xr.Dataset(
    data_vars={
        "a": (("x", "y"), np.random.randn(7, 3)),
        "b": (("z", "x"), np.random.randn(2, 7))
    },
    coords={
        "x": np.arange(6, 13),
        "y": np.arange(3),
        "z": np.arange(3, 5)
    }
)
# NetCDF 파일로 저장
ds2.to_netcdf(datadir2/"ds2.nc")
```

#### DataArray 저장
```python
# DataArray를 개별 NetCDF 파일로 저장
ds1.a.to_netcdf(datadir2/"da1.nc")
```

### 파일 열기
- `xr.open_dataset()`: 단일 NetCDF/Zarr 파일 열기
- `xr.open_mfdataset()`: 여러 파일을 하나의 Dataset으로 열기

---

## 요약
- **Xarray**는 레이블된 다차원 배열을 다루는 강력한 도구
- **DataArray**는 단일 변수를 다루는 기본 구조
- **Dataset**은 여러 DataArray를 모아 놓은 컨테이너
- pandas, NetCDF, Zarr 등 다른 형식과의 변환이 용이함
- 차원, 좌표, 속성을 통해 데이터에 풍부한 메타데이터 추가 가능

초안 정리 후 claude가 최종 정리 하였습니다.