아래는 사용자의 쿼리에 따라, BatteryDataTool.py의 로직을 참고하여 충방전기 데이터를 로드하는 구현 계획을 작성한 것입니다. 계획은 단계별로 구성되며, 대용량 데이터(200MB 이상)를 다루기 위해 메모리 효율성을 강조합니다. BatteryDataTool.py의 핵심 로직(check_cycler로 유형 구분, toyo_* 함수로 Toyo 데이터 처리, pne_* 함수로 PNE 데이터 처리)을 기반으로 하되, 대용량 데이터 처리를 위해 추가 최적화(Pandas chunksize, Dask 사용, 데이터 타입 최적화 등)를 제안합니다.

ipynb 파일 형식으로 아래 계획을 구현하라. UI 기능은 제거하라.

### 1. 전체 구현 계획 개요
- **목적**: 주어진 path(디렉토리)에서 충방전기 데이터를 로드하고, 유형(Toyo vs PNE)에 따라 cycle 데이터(전체 사이클 요약)와 profile 데이터(상세 프로필)를 처리. 여러 파일 병합 지원. 로드 후 시각화(matplotlib 기반) 준비.
- **입력**: path (e.g., "D:/battery_data_folder") – 디렉토리 경로. 내부에 CSV/log 파일 또는 Restore 서브디렉토리가 있음.
- **출력**: 
  - Pandas DataFrame (cycle_df: 사이클 요약, profile_df: 상세 프로필).
  - 병합된 데이터 프레임 (여러 사이클/파일 병합 시).
  - 시각화: matplotlib으로 그래프 생성 (e.g., graph_cycle, graph_profile 함수 호출).
- **대용량 데이터 처리 원칙**:
  - 메모리 누락 방지: 파일을 chunk 단위로 읽음 (Pandas chunksize 또는 Dask 사용).
  - 효율성: 필요한 column만 로드, 데이터 타입 최적화 (e.g., float32/int32), 불필요 데이터 필터링.
  - 병렬 처리: Dask로 대용량 파일 병합/처리 (멀티코어 활용).
  - 에러 핸들링: 파일 누락/인코딩 오류 시 예외 처리 (e.g., try-except, 로그 출력).
- **라이브러리**: Pandas, NumPy, Matplotlib (BatteryDataTool.py 기반), Dask (대용량 최적화), os/re (파일 탐색).
- **예상 성능**: 200MB 파일 기준, chunksize=10^5로 로드 시 메모리 50-100MB 이내, 로드 시간 10-30초 (하드웨어 의존).

### 2. 단계별 구현 계획
#### 단계 1: 입력 path 확인 및 충방전기 유형 구분
- **로직 참고**: BatteryDataTool.py의 `check_cycler` 함수 (Pattern 폴더 유무로 PNE vs Toyo 구분).
- **구현 세부**:
  - path가 디렉토리인지 확인 (os.path.isdir). 아니면 에러 메시지 출력 (e.g., err_msg 함수 호출).
  - 유형 구분:
    - if os.path.isdir(path + "/Pattern"): PNE (True 반환).
    - else: Toyo (False 반환).
  - 추가: path 내 파일 목록 캐싱 (os.listdir) – 대용량 디렉토리 시 glob 사용으로 효율화.
  - 에러 핸들링: path不存在 시 filedialog.askdirectory로 사용자 재입력 유도 (BatteryDataTool.py 스타일).
- **대용량 최적화**: 유형 구분 시 파일 스캔 최소화 (Pattern 폴더만 체크).
- **코드 스케치**:
  ```python
  import os
  import pandas as pd
  import dask.dataframe as dd  # 대용량용
  import matplotlib.pyplot as plt

  def check_cycler(path):
      return os.path.isdir(os.path.join(path, "Pattern"))  # PNE: True, Toyo: False
  ```

#### 단계 2: Toyo 데이터 로드 (유형 == False)
- **로직 참고**: toyo_read_csv, toyo_cycle_import, toyo_Profile_import, toyo_min_cap, toyo_cycle_data 등.
- **Cycle 데이터 로드**:
  - 파일: path + "/capacity.log" (skiprows=0).
  - 처리: pd.read_csv(..., chunksize=100000, usecols=["TotlCycle", "Condition", "Cap[mAh]", ...]) – chunk로 읽어 메모리 절약.
  - min_cap 산정: toyo_min_cap (파일명에서 용량 추출 or 첫 사이클 전류 기반).
  - 데이터 처리: toyo_cycle_data (충방전 병합, DCIR 계산, 효율/용량 ratio 계산) – Dask로 병렬 적용 (dd.apply).
  - 결과: cycle_df (Dchg, Eff, Temp 등 컬럼).
- **Profile 데이터 로드**:
  - 파일: path + "/%06d" % cycle (skiprows=3).
  - 처리: toyo_Profile_import (chunk 로드, usecols=["PassTime[Sec]", "Voltage[V]", "Current[mA]", ...]).
  - 유형별: toyo_step_Profile_data (step charge), toyo_rate_Profile_data (율별), toyo_chg/dchg_Profile_data (충/방전), toyo_Profile_continue_data (연속).
  - 용량 산정: 벡터화 연산 (diff/cumsum) – NumPy로 가속.
  - 결과: profile_df (TimeMin, SOC, Vol 등).
- **파일 병합**:
  - 여러 사이클 파일: for cycle in range(start, end): 로드 후 pd.concat (Dask로 지연 연산: dd.concat).
  - 병합 후 필터: df = df[df["Current[mA]"] >= cutoff * min_cap] (메모리 줄임).
- **대용량 최적화**:
  - Dask 사용: dd.read_csv(path + "/*.csv", blocksize="64MB") – 자동 chunking/병렬.
  - 타입 최적화: df.astype({'Voltage[V]': 'float32', 'Current[mA]': 'int32'}).
  - 메모리 해제: del temp_df 후 gc.collect().
- **코드 스케치**:
  ```python
  def load_toyo_cycle(path, min_cap=0, ini_rate=0.2, chk_ir=True):
      raw = dd.read_csv(os.path.join(path, "capacity.log"), blocksize="64MB", usecols=...)  # Dask로 로드
      min_cap = toyo_min_cap(path, min_cap, ini_rate)  # 용량 산정
      processed = toyo_cycle_data(raw, min_cap, ini_rate, chk_ir)  # 처리 로직 적용
      return processed.compute()  # 최종 Pandas로 변환

  def load_toyo_profile(path, ini_cycle, end_cycle, min_cap, cutoff, ini_rate, smooth=0):
      dfs = []
      for cyc in range(ini_cycle, end_cycle + 1):
          file = os.path.join(path, f"%06d" % cyc)
          chunk_df = pd.read_csv(file, chunksize=100000, skiprows=3, usecols=...)
          for chunk in chunk_df:
              processed = toyo_step_Profile_data(chunk, ...)  # 처리
              dfs.append(processed)
      merged = pd.concat(dfs)  # 병합
      return merged
  ```

#### 단계 3: PNE 데이터 로드 (유형 == True)
- **로직 참고**: pne_data, pne_search_cycle.
- **Cycle 데이터 로드**:
  - 파일: path + "/Restore/*.csv" (SaveEndData 파일 검색).
  - 처리: pne_search_cycle로 원하는 사이클 파일 위치 찾기 (start/end 기준 bisect 사용).
  - 데이터: pd.read_csv(..., chunksize=100000, header=None) – column 27(TotlCycle) 등 추출.
- **Profile 데이터 로드**:
  - 파일: path + "/Restore/*.csv" (SaveData 파일).
  - 처리: pne_data (Restore 디렉토리 순회, 여러 파일 병합: pd.concat).
  - 결과: profile_df (Profileraw).
- **파일 병합**:
  - 여러 파일: subfile 목록에서 SaveData 파일만 필터, 순차 병합 (Dask dd.read_csv("*SaveData*.csv")).
  - 사이클 필터: df = df[df[27] >= start & df[27] <= end].
- **대용량 최적화**:
  - Dask: dd.read_csv(path + "/Restore/*.csv", blocksize="64MB", include_path_column=True) – 파일 경로 포함.
  - 필터링: Dask의 query로 메모리 로드 전 필터 (e.g., df = df[df['Condition'] == 1]).
- **코드 스케치**:
  ```python
  def load_pne_cycle(path, start_cycle, end_cycle):
      restore_path = os.path.join(path, "Restore")
      files = [f for f in os.listdir(restore_path) if "SaveEndData" in f and f.endswith(".csv")]
      pos = pne_search_cycle(restore_path, start_cycle, end_cycle)  # 파일 위치 찾기
      raw = dd.read_csv([os.path.join(restore_path, files[i]) for i in range(pos[0], pos[1]+1)], blocksize="64MB", header=None)
      return raw.compute()

  def load_pne_profile(path, ini_cycle):
      restore_path = os.path.join(path, "Restore")
      raw = dd.read_csv(os.path.join(restore_path, "*SaveData*.csv"), blocksize="64MB", header=None)
      processed = pne_data(raw, ini_cycle)  # 처리 및 병합
      return processed.compute()
  ```

#### 단계 4: 데이터 병합 및 후처리
- **병합 로직**: Cycle + Profile 병합 (pd.merge on 'TotlCycle').
- **후처리**: BatteryDataTool.py 기반 – dQdV 산정, 단위 변환 (e.g., PassTime[Sec]/60).
- **대용량**: Dask로 병합 (dd.merge), compute()로 최종 Pandas 변환.
- **에러 핸들링**: 파일 누락 시 skip, 로그 출력 (e.g., print("Missing file: ", file)).

#### 단계 5: 시각화
- **로직 참고**: graph_cycle, graph_profile 등.
- **구현**: 로드된 df로 matplotlib 호출 (e.g., graph_cycle(df['index'], df['Dchg'], ax, ...)).
- **대용량 최적화**: 데이터 샘플링 (e.g., df.sample(frac=0.1)) 또는 downsampling (e.g., resample every 100 rows).
- **코드 스케치**:
  ```python
  def visualize(df_cycle, df_profile):
      fig, ax = plt.subplots()
      graph_cycle(df_cycle['index'], df_cycle['Dchg'], ax, ...)  # Cycle 그래프
      graph_profile(df_profile['TimeMin'], df_profile['Vol'], ax, ...)  # Profile 그래프
      plt.show()
  ```

#### 단계 6: 전체 함수 래퍼 및 테스트
- **메인 함수**:
  ```python
  def load_battery_data(path, start_cycle=1, end_cycle=10, min_cap=0, cutoff=0.05, ini_rate=0.2):
      is_pne = check_cycler(path)
      if not is_pne:
          cycle_df = load_toyo_cycle(path, min_cap, ini_rate)
          profile_df = load_toyo_profile(path, start_cycle, end_cycle, ...)
      else:
          cycle_df = load_pne_cycle(path, start_cycle, end_cycle)
          profile_df = load_pne_profile(path, start_cycle)
      merged_df = pd.merge(cycle_df, profile_df, on='TotlCycle', how='outer')
      visualize(cycle_df, profile_df)
      return merged_df
  ```
- **테스트**: 샘플 path로 로드 시간/메모리 측정 (psutil 사용).
- **배포 고려**: multiprocessing으로 병렬 로드 (e.g., 여러 사이클 병렬 처리).

이 계획을 따르면 BatteryDataTool.py의 로직을 유지하면서 대용량 데이터를 효율적으로 처리할 수 있습니다. 실제 구현 시 Dask 설치(pip install dask)를 권장합니다. 추가 질문 있으시면 말씀해주세요!