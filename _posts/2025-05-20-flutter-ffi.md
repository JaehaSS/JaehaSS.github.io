---
layout: post
title: "Flutter의 FFI"
date: 2025-05-20
description: "Flutter에서 FFI(Foreign Function Interface)를 사용하여 C 네이티브 라이브러리와 통신하는 방법과 예제를 정리한 글입니다."
tags: [FLUTTER]
tistory_id: 61
---

# FFI 란?

두 개의 서로 다른 프로그래밍 언어가 서로 통신하고 호출할 수 있도록 도와주는 라이브러리.
Ex) 요기요에선 성능이 필요한 부분은 Rust같은 언어로 처리

### 사용하는 이유

- 기존에 작성된 네이티브 라이브러리를 재사용하기 위해
- 애플리케이션 특정 부분에서 성능 이점을 위해
- 플랫폼 특정 기능을 사용하기 위해(OpenCV) 지원 플랫폼 웹을 제외한 모든 플랫폼

### 단점

- 타입 호환성
- 메모리 관리
- 오류 처리

## 예제

A, B의 양의 정수의 범위를 정하여 해당 범위만큼의 누적합을 더하는 함수를 예제로 만들어서 보여드리겠습니다.

### 1. ffi 설치

```
flutter pub get ffi
```

### 2. 구현 부분

```
.
├── lib
│   ├── main.dart
│   └── native_service.dart
├── src
│   ├── CMakeLists.txt
│   └── native_add.c
```

### 3. 구현

```
# main.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart'; // For FilteringTextInputFormatter
import 'native_service.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter FFI vs Dart 비교',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const MyHomePage(),
    );
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({super.key});

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  final NativeService _nativeService = NativeService();
  final TextEditingController _controllerStart = TextEditingController();
  final TextEditingController _controllerEnd = TextEditingController();

  String _nativeResult = '네이티브 결과가 여기에 표시됩니다.';
  String _dartResult = 'Dart 결과가 여기에 표시됩니다.';
  String _nativeTime = '';
  String _dartTime = '';

  // Dart에서 범위 덧셈을 수행하는 함수
  int dartSumRange(int start, int end) {
    if (start > end) { return 0; }
    BigInt sum = BigInt.zero;
    for (int i = start; i <= end; i++) {
      sum += BigInt.from(i);
      if (sum > BigInt.from(9223372036854775807) || sum < BigInt.from(-9223372036854775808)) {
        throw RangeError('결과가 int 범위를 벗어났습니다.');
      }
    }
    return sum.toInt();
  }

  void _calculateRangeSum() {
    final int? start = int.tryParse(_controllerStart.text);
    final int? end = int.tryParse(_controllerEnd.text);

    if (start != null && end != null) {
      if (start > end) {
        setState(() {
          _nativeResult = '시작 값은 종료 값보다 클 수 없습니다.';
          _dartResult = '시작 값은 종료 값보다 클 수 없습니다.';
          _nativeTime = '';
          _dartTime = '';
        });
        return;
      }

      // 네이티브 함수 실행 및 시간 측정
      try {
        final stopwatchNative = Stopwatch()..start();
        final sumNative = _nativeService.sumRange(start, end);
        stopwatchNative.stop();
        setState(() {
          _nativeResult = '네이티브: $start + ... + $end = $sumNative';
          _nativeTime = '네이티브 실행 시간: ${stopwatchNative.elapsedMilliseconds} ms';
        });
      } catch (e) {
        setState(() {
          _nativeResult = '네이티브 오류: $e';
          _nativeTime = '';
        });
      }

      // Dart 함수 실행 및 시간 측정
      try {
        final stopwatchDart = Stopwatch()..start();
        final sumDart = dartSumRange(start, end);
        stopwatchDart.stop();
        setState(() {
          _dartResult = 'Dart: $start + ... + $end = $sumDart';
          _dartTime = 'Dart 실행 시간: ${stopwatchDart.elapsedMilliseconds} ms';
        });
      } catch (e) {
        setState(() {
          _dartResult = 'Dart 오류: $e';
          _dartTime = '';
        });
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('FFI vs Dart - 범위 덧셈'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.start,
          children: <Widget>[
            TextField(
              controller: _controllerStart,
              decoration: const InputDecoration(labelText: '시작 정수 (int32)'),
              keyboardType: TextInputType.number,
            ),
            const SizedBox(height: 16),
            TextField(
              controller: _controllerEnd,
              decoration: const InputDecoration(labelText: '종료 정수 (int32)'),
              keyboardType: TextInputType.number,
            ),
            const SizedBox(height: 32),
            ElevatedButton(
              onPressed: _calculateRangeSum,
              child: const Text('범위 덧셈 실행 (네이티브 & Dart)'),
            ),
            const SizedBox(height: 32),
            Text(_nativeResult, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold), textAlign: TextAlign.center),
            Text(_nativeTime, style: const TextStyle(fontSize: 14, color: Colors.grey), textAlign: TextAlign.center),
            const SizedBox(height: 16),
            Text(_dartResult, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold), textAlign: TextAlign.center),
            Text(_dartTime, style: const TextStyle(fontSize: 14, color: Colors.grey), textAlign: TextAlign.center),
          ],
        ),
      ),
    );
  }
}
```

```
# native_service.dart
import 'dart:ffi';
import 'dart:io' show Platform;

typedef NativeSumRangeCSignature = Int64 Function(Int32 start, Int32 end);
typedef NativeSumRangeDartSignature = int Function(int start, int end);

class NativeService {
  late final NativeSumRangeDartSignature _nativeSumRange;

  NativeService() { _loadLibrary(); }

  void _loadLibrary() {
    final DynamicLibrary dylib = _openDynamicLibrary();
    _nativeSumRange = dylib
        .lookup<NativeFunction<NativeSumRangeCSignature>>('native_sum_range')
        .asFunction<NativeSumRangeDartSignature>();
  }

  DynamicLibrary _openDynamicLibrary() {
    if (Platform.isMacOS || Platform.isIOS) { return DynamicLibrary.open('libnative_add.dylib'); }
    if (Platform.isAndroid || Platform.isLinux) { return DynamicLibrary.open('libnative_add.so'); }
    if (Platform.isWindows) { return DynamicLibrary.open('native_add.dll'); }
    throw UnsupportedError('Unsupported platform: ${Platform.operatingSystem}');
  }

  int sumRange(int start, int end) { return _nativeSumRange(start, end); }
}
```

```
# native_add.c
#include <stdint.h>

#if defined(_WIN32)
#define FFI_EXPORT __declspec(dllexport)
#elif defined(__APPLE__) || defined(__MACH__) || defined(__linux__)
#define FFI_EXPORT __attribute__((visibility("default"))) __attribute__((used))
#else
#define FFI_EXPORT
#endif

FFI_EXPORT int64_t native_sum_range(int32_t start, int32_t end) {
    if (start > end) { return 0; }
    int64_t sum = 0;
    for (int32_t i = start; i <= end; ++i) {
        if (i > 0) {
            if (sum > INT64_MAX - i) { return INT64_MAX; }
        } else if (i < 0) {
            if (sum < INT64_MIN - i) { return INT64_MIN; }
        }
        sum += i;
    }
    return sum;
}
```

### 4. gradle 수정

```
...
externalNativeBuild {
    cmake {
        path = file("../../src/CMakeLists.txt")
    }
}
```

### 5. 검증

![image.png](https://blog.kakaocdn.net/dna/attachment/df24bcbb-d875-4875-aa08-464b86828db4/image.png)
