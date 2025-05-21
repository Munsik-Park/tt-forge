# tt-forge 맥북(M3, ARM) 환경 실행 시도 및 문제점 로그

## 1. 로컬(ARM) 환경에서 Python 가상환경 및 패키지 설치
- Python venv 생성 및 활성화
- `pip install torch torchvision timm pillow` 등 패키지 설치
- `pip install`로 forge, tvm wheel(x86_64용) 설치 시도  
  → **오류:** "is not a supported wheel on this platform" (ARM에서 x86_64 wheel 설치 불가)

## 2. Docker Desktop 설치 및 실행
- Docker Desktop(Mac ARM용) 설치 및 실행
- **컨테이너 실행 명령어:**
  ```bash
  docker run --platform linux/amd64 -it -v $(pwd):/workspace ghcr.io/tenstorrent/tt-forge-fe/tt-forge-fe-ird-ubuntu-22-04
  ```
  - `--platform linux/amd64`: x86_64 에뮬레이션 모드로 실행
  - `-v $(pwd):/workspace`: 현재 디렉토리를 컨테이너의 /workspace에 마운트
  → **경고:** "The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)"
- 컨테이너 내에서 `/workspace`로 소스 마운트

## 3. 컨테이너 내에서 Python 가상환경 생성 및 활성화
- 가상환경 생성 및 활성화 방법:
  ```bash
  python3 -m venv tt-forge-env
  source tt-forge-env/bin/activate
  ```
- (가상환경 활성화 후) 필요한 패키지 설치:
  ```bash
  pip install torch torchvision timm pillow
  pip install https://github.com/tenstorrent/tt-forge/releases/download/0.1.0.dev20250422214451/forge-0.1.0.dev20250422214451-cp310-cp310-linux_x86_64.whl
  pip install https://github.com/tenstorrent/tt-forge/releases/download/0.1.0.dev20250422214451/tvm-0.1.0.dev20250422214451-cp310-cp310-linux_x86_64.whl
  ```
- **의존성 충돌:** numpy, timm, decorator, ml-dtypes, jax, jaxlib 등 버전 불일치
- forge가 요구하는 버전으로 재설치 시도  
  → **여전히 의존성 충돌**

## 4. JAX/JAXLIB 관련 문제
- 데모 실행 시 `import forge`에서 내부적으로 `jax` import
- **오류:**  
  ```
  RuntimeError: This version of jaxlib was built using AVX instructions, which your CPU and/or operating system do not support.
  ```
  - x86_64(AVX) 명령어 미지원 환경(ARM, 또는 AVX 없는 x86)에서는 실행 불가

## 5. x86_64 에뮬레이션 모드로 Docker 실행
- 위의 컨테이너 실행 명령어와 동일하게 실행
- 동일하게 forge, tvm, jax 등 설치 및 데모 실행
- **동일 오류:**  
  ```
  RuntimeError: This version of jaxlib was built using AVX instructions, which your CPU and/or operating system do not support.
  ```
  - 에뮬레이션 환경에서도 실제 하드웨어가 AVX를 지원하지 않으면 실행 불가

## 9. 결론 및 요약

### 성공한 부분:
1. **Docker 설치 및 설정** ✓
2. **Python 가상환경 생성** ✓  
3. **pyproject.toml 버전 문제 해결** ✓
4. **모든 Python 패키지 설치** ✓
   - ARM에서 실패했던 x86_64 wheel 설치 성공
   - AVX 명령어 문제 해결됨
5. **forge, torch import 성공** ✓
6. **MLIR 컴파일 성공** ✓
   - MLIR 모듈 생성
   - MLIR 패스 실행
   - Flatbuffer 바이너리 생성

### 한계점:
- **Tenstorrent 하드웨어 부재**: 실제 모델 실행을 위해서는 Wormhole 또는 Blackhole 가속기 카드 필요
- 컴파일은 성공하지만 "Opening user mode device driver" 단계에서 하드웨어 미발견으로 segmentation fault 발생

### ARM vs x86 비교:
- **ARM (이전)**: AVX 명령어 미지원으로 `import forge`, `import jax` 단계에서 실패
- **x86 (현재)**: 모든 import 및 컴파일 성공, 하드웨어 액세스 단계에서만 실패

### 결론:
x86 환경에서 ARM의 AVX 문제는 완전히 해결되었으며, tt-forge 컴파일러의 모든 소프트웨어 스택이 정상 작동함을 확인. 실제 실행을 위해서는 Tenstorrent 하드웨어가 필요한 상황.

echo "로그 파일이 tt-forge-x86-execution-log.md로 저장되었습니다." 