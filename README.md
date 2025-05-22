# PPS(퍼플 페이먼트 시스템) 통합 가이드

## 주요 기능 개요

PPS(퍼플 페이먼트 시스템)는 다음과 같은 주요 기능을 제공합니다:

1. **USDT/TRX 출금 처리**: 고객이 요청한 출금을 안전하게 처리합니다.
2. **텔레그램 승인 시스템**: 설정된 금액 이상의 출금은 관리자의 텔레그램 승인이 필요합니다.
3. **입금 모니터링**: 지정된 지갑 주소로 들어오는 입금을 자동으로 감지합니다.
4. **콜백 알림**: 모든 트랜잭션 상태 변경 시 고객사 시스템에 실시간 알림을 제공합니다.
5. **재시도 메커니즘**: 콜백 전송 실패 시 자동 재시도 로직을 통해 안정적인 통신을 보장합니다.

이 문서는 PPS(퍼플 페이먼트 시스템)와 통합하여 입금 및 출금 이벤트에 대한 콜백을 처리하는 방법을 설명합니다. 콜백 시스템을 통해 고객사 시스템은 실시간으로 트랜잭션 상태 변경을 감지하고 적절한 조치를 취할 수 있습니다.

## API 및 콜백 URL 설정

### API 인증

PPS(퍼플 페이먼트 시스템)의 모든 API 요청은 인증이 필요합니다. 인증을 위해 다음과 같은 헤더를 모든 API 요청에 포함해야 합니다:

```
Authorization: Bearer your-api-key-here
```

- API 키는 서비스 관리자로부터 발급받을 수 있습니다.
- API 키는 절대 노출되지 않도록 안전하게 관리해야 합니다.
- 인증 실패 시 HTTP 상태 코드 401(Unauthorized)이 반환됩니다.

### 제공되는 API

PPS(퍼플 페이먼트 시스템)는 다음과 같은 API를 제공합니다:

1. **출금 요청 API**
   - 엔드포인트: `POST /api/withdrawal/request`
   - 설명: USDT 또는 TRX 출금을 요청합니다.
     - `currency_type`이 "KRW"인 경우 자동으로 현재 환율을 적용하여 출금합니다.
   - 필수 파라미터:
     - `user_wallet`: TRON 지갑 주소
     - `amount`: 출금 금액
     - `request_id`: 요청 고유 ID
     - `currency_type`: 통화 타입 ("KRW" 또는 "crypto")
   - 선택 파라미터:
     - `token_type`: 토큰 타입 ("USDT" 또는 "TRX", 기본값: "USDT")
     - `user_name`: 사용자 이름 (텔레그램 승인 메시지에 표시됨)
   - 요청 예시 (원화 금액):
     ```json
     {
       "user_wallet": "TRX-wallet-address",
       "amount": "14000",
       "token_type": "USDT",
       "currency_type": "KRW",
       "request_id": "your-unique-request-id"
     }
     ```
   - 요청 예시 (암호화폐 금액):
     ```json
     {
       "user_wallet": "TRX-wallet-address",
       "amount": "10.5",
       "token_type": "USDT",
       "currency_type": "crypto",
       "request_id": "your-unique-request-id"
     }
     ```

2. **트랜잭션 상태 조회 API**
   - 엔드포인트: `GET /api/transaction/status`
   - 설명: 특정 트랜잭션(출금/입금)의 상태를 조회합니다.
   - 파라미터 (두 가지 중 하나는 필수):
     - `transaction_id`: 트랜잭션 ID
     - `tx_hash`: 블록체인 트랜잭션 해시
   - 예시: 
     - `GET /api/transaction/status?transaction_id=1234-5678-90ab-cdef`
     - `GET /api/transaction/status?tx_hash=0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef`
   - 응답 예시:
     ```json
     {
       "success": true,
       "data": {
         "transaction_id": "1234-5678-90ab-cdef",
         "status": "completed",
         "amount": "10.5",
         "token_type": "USDT",
         "tx_hash": "0x1234...",
         "user_wallet": "TRX-wallet-address",
         "type": "withdrawal",
         "currency_type": "KRW",
         "original_amount": "14000.00",
         "exchange_rate": "1333.333333",
         "created_at": "2025-05-20T06:00:00.000Z",
         "updated_at": "2025-05-20T06:05:00.000Z"
       }
     }
     ```

## 서비스 연동 정보

### 서비스 관리자에게 전달해야 할 정보

서비스 연동을 위해 다음 정보를 서비스 관리자에게 전달해야 합니다:

1. **콜백 URL**: 트랜잭션 상태 변경 시 알림을 받을 엔드포인트 URL
   ```
   https://your-domain.com/api/pps-callback
   ```

2. **고객사 지갑 주소**: TRON 네트워크 지갑 주소
3. **고객사 지갑 private key**: TRON 네트워크 지갑 private key
4. **텔레그램 승인 관련 정보** (텔레그램 승인 기능 사용 시):
   - 텔레그램 채팅 ID: 텔레그램 봇을 통해 확인한 ID
   - 임계값: 승인이 필요한 출금 금액 임계값 (USDT)
   - 승인 비밀번호: 텔레그램 승인 시 사용할 비밀번호

## 텔레그램 승인 기능

### 텔레그램 봇 설정 방법

텔레그램 승인 기능을 사용하려면 다음 절차를 따라주세요:

1. [퍼플테터출금승인봇](https://t.me/purple_usdt_transaction_bot) 채팅을 시작합니다.
2. "시작" 혹은 "start" 버튼을 눌러 채팅을 시작합니다.
3. 확인된 채팅 ID를 서비스 관리자에게 전달합니다.

### 텔레그램 승인 프로세스

임계값 이상의 출금 요청이 발생하면 다음과 같은 프로세스가 진행됩니다:

1. 텔레그램 봇을 통해 승인 요청 메시지가 전송됩니다.
2. 승인 요청 메시지에는 출금 정보(주소, 금액, 토큰 유형 등)가 포함됩니다.
3. 관리자는 "승인" 또는 "거부" 버튼을 클릭하여 요청을 처리할 수 있습니다.
4. 승인 시 비밀번호 입력이 필요할 수 있습니다.

## 콜백 데이터 형식 및 호출 시점

### 데이터 필드 설명

- **transaction_id**: 트랜잭션 고유 식별자 (UUID 형식)
- **tx_hash**: 블록체인 트랜잭션 해시
- **status**: 트랜잭션 상태 ('pending', 'processing', 'completed', 'failed', 'awaiting_approval', 'rejected')
- **amount**: 출금 금액 (암호화폐 단위)
- **user_wallet**: 사용자 지갑 주소
- **type**: 트랜잭션 유형 ('deposit' 또는 'withdrawal')
- **token_type**: 토큰 유형 ('USDT' 또는 'TRX')
- **currency_type**: 통화 유형 ('KRW' 또는 'crypto')
- **original_amount**: 원래 요청 금액 (원화일 경우)
- **exchange_rate**: 요청 시점의 환율
- **timestamp**: 콜백 전송 시간 (ISO 8601 형식)
- **user_wallet**: 사용자 지갑 주소
- **type**: 트랜잭션 타입 (withdrawal, deposit)
- **token_type**: 토큰 타입 (USDT, TRX)
- **timestamp**: 트랜잭션 처리 시간 (ISO 8601 형식)
- **error**: 실패 시 오류 메시지 (실패 시에만 포함)

### 1. 출금 콜백

출금 요청이 처리되면 다음 형식의 데이터가 고객사 시스템으로 전송됩니다:

```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "tx_hash": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "status": "completed",
  "amount": "10.5",
  "user_wallet": "TRX-wallet-address",
  "type": "withdrawal",
  "token_type": "USDT",
  "currency_type": "KRW",
  "original_amount": "14000.00",
  "exchange_rate": "1333.333333",
  "timestamp": "2025-05-20T06:00:00.000Z"
}
```

**호출 시점**:
- **출금 승인 후**: 텔레그램 관리자가 출금을 승인하고 블록체인에 트랜잭션이 기록된 후
- **출금 거부 시**: 텔레그램 관리자가 출금을 거부했을 때
- **자동 승인 시**: 설정된 임계값 미만의 금액은 자동 승인 후 블록체인에 트랜잭션이 기록된 후

**출금 콜백 처리 주의사항**:

출금 처리를 안전하게 구현하기 위해 다음 절차를 따르는 것을 권장합니다:

1. 사용자로부터 출금 요청 받기
2. 사이트 머니 선차감 처리 (사용자 계정에서 출금 금액 차감)
3. 출금 요청 API 호출
4. 콜백 처리:
   - 출금 성공 시: 차감된 머니 확정 처리
   - 출금 실패 시: 차감된 머니 복구 처리

이러한 방식으로 처리하면 시스템 오류나 네트워크 문제로 인한 중복 처리를 방지할 수 있습니다.

실패 시에는 추가적인 오류 정보가 포함됩니다:

```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "tx_hash": "efe0499a376894b91a27f89e976334e59beb881bd7d41af07701a187de14ac61",
  "status": "failed",
  "amount": "10.5",
  "user_wallet": "TRX-wallet-address",
  "type": "withdrawal",
  "token_type": "USDT",
  "currency_type": "KRW",
  "original_amount": "14000.00",
  "exchange_rate": "1333.333333",
  "timestamp": "2025-05-20T06:00:00.000Z",
  "error": "텔레그램 관리자 @username에 의해 거부됨 (2025. 5. 20. 오전 6:00:00)"
}
```

### 2. 입금 콜백

입금이 감지되고 확인되면 다음 형식의 데이터가 고객사 시스템으로 전송됩니다:

```json
{
  "transaction_id": "c43fb9c0-f172-4c55-9332-d7aa6b130c57",
  "tx_hash": "199ac672a38539ac48b15275f4566d54b37222b8380ad30984ffd1220c349ade",
  "status": "completed",
  "amount": "5.0",
  "user_wallet": "TRX-wallet-address",
  "type": "deposit",
  "token_type": "USDT",
  "currency_type": "KRW",
  "original_amount": "14000.00",
  "exchange_rate": "1333.333333",
  "timestamp": "2025-05-20T03:22:52.000Z"
}
```

입금 실패 시에는 다음과 같은 데이터가 전송됩니다:

```json
{
  "transaction_id": "c43fb9c0-f172-4c55-9332-d7aa6b130c57",
  "tx_hash": "199ac672a38539ac48b15275f4566d54b37222b8380ad30984ffd1220c349ade",
  "status": "failed",
  "amount": "5.0",
  "user_wallet": "TRX-wallet-address",
  "type": "deposit",
  "token_type": "USDT",
  "currency_type": "KRW",
  "original_amount": "14000.00",
  "exchange_rate": "1333.333333",
  "timestamp": "2025-05-20T03:22:52.000Z",
  "error": "트랜잭션을 찾을 수 없음"
}
```

**호출 시점**:
- **입금 확인 후**: 블록체인에서 입금 트랜잭션이 감지되고 설정된 확인 수(기본 20회)에 도달한 후

## 콜백 응답 요구사항

고객사 시스템은 콜백을 수신한 후 HTTP 상태 코드 200으로 응답해야 합니다. 응답 본문은 필요하지 않으며, 빈 응답(empty response)도 유효합니다.

## 재시도 메커니즘

콜백 전송이 실패하거나 고객사 시스템이 적절한 응답을 반환하지 않는 경우, PPS(퍼플 페이먼트 시스템)는 다음과 같은 재시도 정책을 따릅니다:

- 초기 실패 후 10초 후 첫 번째 재시도
- 이후 기하급수적인 백오프로 최대 3회 재시도 (30초, 1분, 2분)

## 콜백 처리 예제 코드

### PHP 예제

```php
<?php
// PPS(퍼플 페이먼트 시스템) 콜백 처리 스크립트

// 콜백 데이터 받기
$callbackData = json_decode(file_get_contents('php://input'), true);

// IP 화이트리스팅 확인 (예시)
$allowedIPs = ['123.45.67.89', '98.76.54.32']; // PPS(퍼플 페이먼트 시스템) 서버 IP 주소를 입력하세요
if (!in_array($_SERVER['REMOTE_ADDR'], $allowedIPs)) {
    http_response_code(403);
    exit('Unauthorized IP');
}

// 트랜잭션 타입 확인
if (isset($callbackData['type']) && isset($callbackData['status'])) {
    if ($callbackData['type'] === 'withdrawal') {
        if ($callbackData['status'] === 'completed') {
            // 출금 성공 처리
            handleSuccessfulWithdrawal($callbackData);
        } else {
            // 출금 실패 처리
            handleFailedWithdrawal($callbackData);
        }
    } else if ($callbackData['type'] === 'deposit') {
        // 입금 처리
        handleDeposit($callbackData);
    }
}

// 성공 응답 반환 (중요: 200 응답 코드)
http_response_code(200);
exit;

/**
 * 출금 성공 처리 함수
 */
function handleSuccessfulWithdrawal($data) {    
    $transactionId = $data['transaction_id'];    
    $amount = $data['amount']; // 달러
    $originalAmount = $data['original_amount']; // 원화
    $exchangeRate = $data['exchange_rate'];
    $userWallet = $data['user_wallet'];    

    // 예시: 사용자 출금 상태 업데이트 및 선차감된 머니 확정 처리    
}

/**
 * 출금 실패 처리 함수
 */
function handleFailedWithdrawal($data) {
    // 데이터베이스에 출금 실패 기록
    $transactionId = $data['transaction_id'];    
    $amount = $data['amount']; // 달러    
    $originalAmount = $data['original_amount']; // 원화
    $exchangeRate = $data['exchange_rate'];
    $errorMessage = $data['error'] ?? '알 수 없는 오류';
    
    // 예시: 사용자 출금 상태 업데이트 및 머니 복구 처리    
}

/**
 * 입금 처리 함수 - 항상 원화로 사용자 잔액 업데이트
 */
function handleDeposit($data) {
    // 데이터베이스에 입금 기록
    $transactionId = $data['transaction_id'];
    $userWallet = $data['user_wallet'];        
    $amount = $data['amount']; // 달러    
    $originalAmount = $data['original_amount']; // 원화
    $exchangeRate = $data['exchange_rate'];
    $status = $data['status'];
    
    // 실패한 입금인지 확인
    if ($status === 'failed') {
        $errorMessage = $data['error'] ?? '알 수 없는 오류';
        logDepositFailure($transactionId, $userWallet, $amount, $originalAmount, $exchangeRate, $errorMessage);
        return false;
    }
    
    
    
    // 원화 기준으로 사용자 잔액 업데이트
    try {
        // 1. 입금 로그 기록
        $logId = logDeposit(
            $transactionId, 
            $userWallet, 
            $amount, 
            $originalAmount, 
            $exchangeRate
        );
        
        // 2. 사용자 DB에서 현재 잔액 조회
        $user = getUserByWallet($userWallet);
        if (!$user) {
            throw new Exception("사용자를 찾을 수 없습니다: {$userWallet}");
        }
        
        // 3. 현재 원화 잔액에 입금된 원화 값 추가
        $currentBalance = $user['balance'] ?? 0;
        $newBalance = $currentBalance + $originalAmount;
        
        // 4. 사용자 원화 잔액 업데이트
        $result = updateUserBalance($userWallet, $newBalance);
        
        if (!$result) {
            throw new Exception("사용자 잔액 업데이트 실패");
        }        
        
        // 5. 성공 응답
        return true;
        
    } catch (Exception $e) {
        // 오류 발생 시 로그 기록
        logError(
            "deposit_error", 
            $transactionId, 
            $e->getMessage(), 
            [
                'user_wallet' => $userWallet,
                'amount' => $amount,
                'original_amount' => $originalAmount
            ]
        );
        
        // 오류 발생 시에도 성공 응답을 보내서 콜백 재시도 방지
        // 내부적으로 오류를 처리하고 수동으로 해결해야 함
        return true;
    }
}
```

## 보안 고려사항

콜백 시스템을 안전하게 사용하기 위해 다음 사항을 고려하세요:

1. **IP 화이트리스팅**: PPS(퍼플 페이먼트 시스템) 서버의 IP를 화이트리스팅하여 허가된 IP에서만 콜백을 받도록 설정하세요.
   - 프로덕션 환경에서 사용할 IP 주소는 PPS(퍼플 페이먼트 시스템) 관리자에게 문의하세요.

2. **HTTPS 사용**: 모든 통신은 HTTPS를 통해 암호화되어야 합니다.
   - 콜백 URL은 반드시 https://로 시작해야 합니다.

3. **콜백 ID 확인**: 모든 콜백 요청에는 `X-Callback-ID` 헤더가 포함되어 있습니다.
   - 이 값을 기록하여 중복 처리를 방지하세요.
   - 동일한 콜백 ID가 여러 번 전송될 수 있으니 주의하세요 (재시도 메커니즘).

4. **오류 처리**: 콜백 처리 중 오류가 발생해도 HTTP 200 응답을 반환하고, 내부적으로 오류를 기록하여 나중에 처리하세요.
   - 이는 불필요한 재시도를 방지하고 오류 추적을 용이하게 합니다.

5. **중복 처리 방지**: 동일한 트랜잭션 ID에 대한 중복 처리를 방지하는 로직을 구현하세요.
   - 트랜잭션 ID를 기준으로 이미 처리된 트랜잭션인지 확인하는 로직 추가:




## 문제 해결

### 일반적인 문제

1. **콜백이 수신되지 않음**
   - 등록된 콜백 URL이 올바른지 확인
   - 방화벽 설정 확인
   - PPS(퍼플 페이먼트 시스템) 관리자에게 문의

2. **중복 콜백 수신**
   - 멱등성 처리 구현 (transaction_id를 기준으로 중복 처리 방지)
   - 재시도 메커니즘으로 인한 정상적인 동작일 수 있음

