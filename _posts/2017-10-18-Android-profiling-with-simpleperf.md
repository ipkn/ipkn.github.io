---
layout: post
title:  "안드로이드 네이티브 프로파일링"
date:   2017-10-18 18:20:50 +0900
categories: Android profiling native simpleperf
---

기본 유니티 프로파일러는 딥 프로파일러로 콜스택 뎁스가 깊으면 심하게 부정확해지고 성능 영향도 심하여 복잡한 성능 문제가 있는 게임에는 사용할 수 없다.  
유니티를 il2cpp 백엔드로 빌드시 네이티브 프로파일링이 가능해진다. 이를 이용하여 성능 측정에 활용해보자.

## simpleperf
안드로이드 ndk에 일정 이후 버전부터 simpleperf 라는 프로파일링용 도구를 제공한다. 단 안드로이드 일정 이상 버전에서만 동작하며 하드웨어 지원도 요구한다. 하드웨어 지원만 한다면, simpleperf가 제공하는 툴 (파이썬 커맨드라인 툴)이 동작하지 않는 버전의 안드로이드 폰에서도 성능 측정을 할 수 있다.

### 다운로드
다음 링크에서 안내된 대로 git clone하여 받을 수 있다: [https://android.googlesource.com/platform/prebuilts/simpleperf/](https://android.googlesource.com/platform/prebuilts/simpleperf/)

## 사용 준비
먼저 native 빌드로 빌드한 apk를 설치해 둔다. 심볼을 strip 하는 옵션을 끌 수 있다면 꺼 둔다. 설명을 위해 앱의 이름을 com.example.app 이라고 가정하자. 여기선 최대한 유저와 비슷한 상황을 만들기 위해 루팅을 안한다고 가정한다. 그리고 안드로이드 SDK, NDK, USB로 연결 및 USB 디버깅 옵션 설정은 완료 되어있다고 가정한다.

일단 simpleperf를 com.example.app 폴더 안으로 설치할 필요가 있다. 이는 다음 명령으로 수행할 수 있다.

    adb push bin/simpleperf /data/local/tmp
    adb shell run-as com.example.app cp /data/local/tmp/simpleperf .
    adb shell rm /data/local/tmp/simpleperf

유니티의 경우 심볼 파일이 따로 생성되며 il2cpp 빌드 후 일정 시간이 지나면 해당 파일이 삭제되기 때문에 분석을 위해 복사해두자.

    cp <UnityProjectPath>/Temp/StagingArea/libs/armeabi-v7a/libil2cpp.so.debug libil2cpp.so

여기까지 완료 했으면 simpleperf를 통해 성능을 측정할 준비가 되었다.

## 성능 정보 기록

성능 정보 기록시 pid 정보가 필요하다. (안드로이드 버전에 따라 앱이름으로 할 수 있으나, 테스트에 사용한 폰에서 동작하지 않았다.)

pid를 얻기 위한 쉘스크립트를 따로 작성하여 활용하였다.

`getpid.sh`:

	adb shell ps|grep "\bcom.example.app\b"|awk '{print $2}'

`record.sh`:

	adb shell run-as com.example.app ./simpleperf record -p `./getpid.sh` -g --symfs .

데이터를 수집하면 perf.data에 쌓이게 된다. record 명령을 실행한 시점부터 Ctrl+C나 앱 종료등으로 멈출때까지 콜스택을 샘플링하여 수집한다. 단, 정확한 원인은 모르겠으나 CPU 사용량이나 메모리 사용량이 높은 경우 수집에 실패하는 경우가 있다. 반복해서 실행하여 수집될까지 (eprf.data 파일의 사이즈가 증가하는지 확인) 실행해보는 수 밖에 없다.

자세한 record 명령 옵션에 대해선 simpleperf 매뉴얼을 참고하라.

## 기록된 정보 추출

아래 명령으로 수집된 데이터를 텍스트 형태로 추출할 수 있다.

	adb shell run-as com.example.app ./simpleperf report -n -g --symfs . --full-callgraph > perf.report

자세한 report 명령 옵션에 대해선 simpleperf 매뉴얼을 참고하라.

## 심볼 변환 (optional)

simpleperf에 --symfs . 등으로 옵션을 주면 심볼 위치를 지정할 수 있다고 매뉴얼에 나와 있으나 아직 제대로 매칭되어 동작하는데 성공하지 못했다. 혹시 동작하는 방법을 찾는 다면 알려주길 바란다.

심볼 매칭이 안된 경우 프로파일링 결과에 주소 값으로만 정보가 남게된다. 이를 아까 복사해둔 심볼 파일을 이용하여 함수이름으로 변환하는 과정을 거쳤다.

	import subprocess
	import bisect
	import shutil
	syms = subprocess.check_output(['nm', 'libil2cpp.so']).strip().split('\n')
	syms = [(x[:8], x[11:]) for x in syms]
	syms.sort()

	f = open('perf.report.conv', 'w')

	for lidx, line in enumerate(open('perf.report')):
			if lidx % 10000 == 0:
					print lidx
			if 'il2cpp.so[' in line:
					idx = line.find('il2cpp.so[+')
					idx2 = line.find(']', idx)
					addr = line[idx+len('il2cpp.so[+'):idx2].zfill(8)
					sidx = bisect.bisect(syms, (addr, ''))-1
					result = syms[sidx][1]+ '[+'+hex(int(addr,16)-int(syms[sidx][0], 16))+']'
					line = line[:idx] + result + line[idx2+1:]
			f.write(line)

## 측정 결과 확인하기

변환된 결과물을 텍스트 에디터로 열어서 눈으로 확인할 수도 있고, simpleperf에 포함된 툴로 GUI로 살펴볼 수 있다.

simpleperf 다운받은 폴더의 report.py를 이용하여 `python report.py <변환된 파일>` 형태로 실행하면 측정 결과를 확인할 수 있다.

다만 오픈소스로 만들어진 툴이라 성능이 나쁘고 살펴보기 불편하다. 그외에 FlameGraph 등을 활용할 수 있다. 다음 링크를 확인하라: [https://github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph)
