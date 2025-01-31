## 프로젝트 소개
음식 배달 서비스 cover<br>
해당 서비스 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전 단계를 커버할 수 있도록 구성한 예제로 제시되는 평가포인트에 맞춰 개발 진행되었습니다.
 - 평가포인트 : https://workflowy.com/s/c10811dfdb67/UhOZB2crKOhNPUYp#/fc30a50ea43b
## Table of contents
- [서비스 시나리오](#서비스-시나리오)
- [분석/설계](#분석/설계)
---
### 서비스 시나리오
배달의 민족 커버하기 - https://1sung.tistory.com/106

- 기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다
2. 고객이 결제한다
3. 주문이 되면 주문 내역이 입점상점주인에게 전달된다
4. 상점주인이 확인하여 요리해서 배달 출발한다
5. 고객이 주문을 취소할 수 있다
6. 주문이 취소되면 배달이 취소된다
7. 고객이 주문상태를 중간중간 조회한다
8. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다

- 비기능적 요구사항
1. 트랜잭션<br>
결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다 Sync 호출
2. 장애격리<br>
상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
3. 성능<br>
고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS
배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven

---
### 분석/설계
---

#### Saga(Pub/Sub)

SAGA 패턴이란 마이크로서비스들끼리 이벤트를 주고 받아 특정 마이크로서비스에서의 작업이 실패하면 이전까지의 작업이 완료된 마이크서비스들에게 보상 (complemetary) 이벤트를 소싱함으로써 분산 환경에서 원자성(atomicity)을 보장하는 패턴

- 주문이 발생할 경우(Order Service - ordered(event)) publish -> 주문 목록이 업데이트 된다. (Store - UpdateOrderList(Polish)) Subscribe

- 주문이 취소될 경우(Order Service - ordercanceled(event)) publish -> 주문결제가 취소된다. (Payment - CancelPayment(Polish)) Subscribe

<br>

- 주문 생성

<img width="960" src="https://user-images.githubusercontent.com/115772322/197435954-fa78862f-d1d2-4182-b40e-ba3b2b681d26.png">

<br>

- 주문 취소

<img width="665" src="https://user-images.githubusercontent.com/115772322/197435970-62094945-e7f8-41b9-a25a-ae1c16983e1f.png">

<br>

- Kafka에 접속하여 이벤트 확인

<img width="1186" src="https://user-images.githubusercontent.com/115772322/197435566-e66e34ec-7267-4c46-a110-c1bc52610b90.png">

 

---

#### CQRS

주문 발생, 결제 승인, 결제 취소, 배달 시작, 배달 취소 시 고객은 OrderInfo(ReadModel)을 통해 주문상태를 중간중간 조회할 수 있다.

<br>

비동기식으로 처리되어 이벤트 기반의 Kafka를 통해 처리되어 별도 Table에 관리한다.

<br>

- Read Model CRUD 상세설계

![image](https://user-images.githubusercontent.com/115772322/197447342-0e106402-97d8-4603-a94c-475aa67ee924.png)

```

    @StreamListener(KafkaProcessor.INPUT)

    public void whenOrdered_then_CREATE_1 (@Payload Ordered ordered) {

        try {

 

            if (!ordered.validate()) return;

 

            // view 객체 생성

            OrderInfo orderInfo = new OrderInfo();

            // view 객체에 이벤트의 Value 를 set 함

            orderInfo.setId(ordered.getId());

            orderInfo.setOrderStatus("Ordered");

            // view 레파지 토리에 save

            orderInfoRepository.save(orderInfo);

 

        }catch (Exception e){

            e.printStackTrace();

        }

    }

 

```

```

    @StreamListener(KafkaProcessor.INPUT)

    public void whenOrderCanceled_then_UPDATE_1(@Payload OrderCanceled orderCanceled) {

        try {

            if (!orderCanceled.validate()) return;

                // view 객체 조회

            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(orderCanceled.getId());

 

            if( orderInfoOptional.isPresent()) {

                 OrderInfo orderInfo = orderInfoOptional.get();

            // view 객체에 이벤트의 eventDirectValue 를 set 함

                orderInfo.setOrderStatus("OrderCanceled");   

                // view 레파지 토리에 save

                 orderInfoRepository.save(orderInfo);

                }

 

 

        }catch (Exception e){

            e.printStackTrace();

        }

    }

```   

```

    @StreamListener(KafkaProcessor.INPUT)

    public void whenPaymentCanceled_then_UPDATE_2(@Payload PaymentCanceled paymentCanceled) {

        try {

            if (!paymentCanceled.validate()) return;

                // view 객체 조회

            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(Long.valueOf(paymentCanceled.getOrderId()));

 

            if( orderInfoOptional.isPresent()) {

                 OrderInfo orderInfo = orderInfoOptional.get();

            // view 객체에 이벤트의 eventDirectValue 를 set 함

                orderInfo.setOrderStatus("PaymentCanceled");   

                // view 레파지 토리에 save

                 orderInfoRepository.save(orderInfo);

                }

 

 

        }catch (Exception e){

            e.printStackTrace();

        }

    }

```

```

    @StreamListener(KafkaProcessor.INPUT)

    public void whenPaymentApproved_then_UPDATE_4(@Payload PaymentApproved paymentApproved) {

        try {

            if (!paymentApproved.validate()) return;

                // view 객체 조회

            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(Long.valueOf(paymentApproved.getOrderId()));

 

            if( orderInfoOptional.isPresent()) {

                 OrderInfo orderInfo = orderInfoOptional.get();

            // view 객체에 이벤트의 eventDirectValue 를 set 함

                orderInfo.setOrderStatus("PaymentApproved");   

                // view 레파지 토리에 save

                 orderInfoRepository.save(orderInfo);

                }

 

 

        }catch (Exception e){

            e.printStackTrace();

        }

    }

```

```

    @StreamListener(KafkaProcessor.INPUT)

    public void whenDeliveryStarted_then_UPDATE_5(@Payload DeliveryStarted deliveryStarted) {

        try {

            if (!deliveryStarted.validate()) return;

                // view 객체 조회

            Optional<OrderInfo> orderInfoOptional = orderInfoRepository.findById(Long.valueOf(deliveryStarted.getOrderId()));

 

            if( orderInfoOptional.isPresent()) {

                 OrderInfo orderInfo = orderInfoOptional.get();

            // view 객체에 이벤트의 eventDirectValue 를 set 함

                orderInfo.setOrderStatus("DeliveryStarted");   

                // view 레파지 토리에 save

                 orderInfoRepository.save(orderInfo);

                }

 

 

        }catch (Exception e){

            e.printStackTrace();

        }

    }

```

 

