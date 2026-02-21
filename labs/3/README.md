# Лабораторная работа №3

**Тема:** Использование принципов проектирования на уровне методов и классов

**Цель работы:** Получить опыт проектирования и реализации модулей с использованием принципов KISS, YAGNI, DRY, SOLID и др.

## Диаграмма контейнеров

![Диаграмма контейнеров](/labs/3/containers.png)

## Диаграмма компонентов

![Диаграмма компонентов](/labs/3/components.png)

## Диаграмма последовательностей

![Диаграмма последовательностей](/labs/3/sequence.png)

## Модель БД

![ERD](/labs/3/erd.png)

На диаграмме представлена только часть таблиц из базы данных. Все эти таблицы относятся к процессу оформления подписки. У пользователя может быть несколько подписок, а также при покупке подписка сохраняет свои тарифы, чтобы в будущем, при добавлении новых тарифов, старые подписки сохраняли прежнюю стоимость.

## Применение основных принципов разработки

### YAGNI
Показать использования этого принципа нельзя, но при отправлении изменения в репозиторий, все неиспользуемые участки кода подсвечиваются IDE и удаляются разработчиком

### KISS
```kotlin
fun disableAutoRenew(subscriptionId: UUID) {
    logger.info { "Subscription $subscriptionId auto renew disabled" }
    subscriptionRepository.setEnabledAutoPayment(subscriptionId, false)
}
```
Это функция отключения авто продления подписки, что оно буквально и делает, без лишних движений

### DRY
```kotlin
private fun sendToKafka(message: ClientMessage) {
    kafkaTemplate.send(CLIENT_TOPIC, message.id.toString(),message)
}

fun disableSubscription(subId: UUID) {
    log.info { "Sent disable subscription message to kafka with subId=$subId" }
    sendToKafka(
        ClientMessage(
            type = ActionType.UPDATE,
            id = subId,
            isEnabled = false,
            expires = null,
        )
    )
}

fun deleteSubscription(subId: UUID) {
    log.info { "Sent delete subscription message to kafka with subId=$subId" }
    sendToKafka(
        ClientMessage(
            type = ActionType.DELETE,
            id = subId,
            isEnabled = false,
            expires = null,
        )
    )
}
```
Часть кода, которая повторяется вынесена в отдельную функцию `sendToKafka`, которая отправляет сообщение в кафку

### SOLID
```kotlin
fun refundOrder(paymentId: String) {
    val order = orderRepository.findByPaymentId(paymentId)?.takeIf { it.status == OrderStatus.PAID }
        ?:throw OrderByPaymentIdNotFoundException(paymentId)

    val paymentService = getPaymentService()

    if (!paymentService.canRefundPayment()) {
        throw RuntimeException("Payment service do not support orders refund")
    }

    paymentService.refundOrder(order.income, paymentId)

    orderRepository.updateStatusByPaymentId(order.paymentId, OrderStatus.REFUND)
}

interface RefundablePaymentService {
    fun canRefundPayment(): Boolean

    fun refundOrder(cost: Int, paymentId: String) {
        throw NotImplementedError("Refund order not implemented yet")
    }
}
```

S - метод отвечает только за возврат и ничего больше
L - реализация payment service может быть какой угодно, если заменить её, то поведение не изменится
I - интерфейс отвечает ровно за возвратные платежи
D - Spring является DI контейнером, который позволяет заменить реализацию payment service

## Дополнительные принципы разработки

### BDUF
Этот принцип использовался для проектирования сервиса connect-service-jobs, поскольку уже были известны функциональные и нефункциональные требования к сервису и нужно было обеспечить их выполнение на этапе проектирования, чтобы не было необходимости изменять код после разработки.

### SoC
Этот принцип используется, поскольку main-service занимается бизнесовой частью, а connect-service занимается частью взаимодействия с впн серверами и подключениями к ним, об этом main-service вообще ничего не знает и знать не должен.

### MVP
На старте проекта была реализована маленькая часть из всей функциональности, которая позволяла бы протестировать работу сервиса. Например, это ручное подтверждение платежей вместо интеграции с платежной системой, отсутствие систем балансировки и автопродления, и т.д.

### PoC
Когда в проект добавлялись платежные системы, был написан мини проект где тестировалось создание платежей и их подтверждение, после чего ым выкатили проведение платежей через платежную систему для администраторов бота, а когда все еще раз првоерили уже на всех пользователей. То есть проверили фичу в несколько этапов.
