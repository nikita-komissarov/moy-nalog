# Неофициальный API клиент [lknpd.nalog.ru](https://lknpd.nalog.ru/) Мой Налог

[![Php version](https://img.shields.io/packagist/php-v/shoman4eg/moy-nalog?style=flat-square)](composer.json)
[![Latest Version](https://img.shields.io/github/release/shoman4eg/moy-nalog.svg?style=flat-square)](https://github.com/shoman4eg/moy-nalog/releases)
[![Total Downloads](https://img.shields.io/packagist/dt/shoman4eg/moy-nalog.svg?style=flat-square)](https://packagist.org/packages/shoman4eg/moy-nalog)
[![Scrutinizer code quality](https://img.shields.io/scrutinizer/quality/g/shoman4eg/moy-nalog/master?style=flat-square)](https://scrutinizer-ci.com/g/shoman4eg/moy-nalog/?branch=master)
[![Packagist License](https://img.shields.io/packagist/l/shoman4eg/moy-nalog?style=flat-square)](LICENSE)
[![Donate](https://img.shields.io/badge/Donate-Tinkoff-yellow?style=flat-square)](https://www.tinkoff.ru/cf/7rZnC7N4bOO)

## Установка

С помощью `composer`

```bash
$ composer require shoman4eg/moy-nalog
```

Так же вам понядобится релизация виртуального пакета [`psr/http-client-implementation`](https://packagist.org/providers/psr/http-client-implementation)

Рекомендуемые реализации `symfony/http-client` или `guzzlehttp/guzzle`

## Использование

### Настройки
```php
// Необходимо выставить часовой пояс для корректного формирования дат в чеках
// Можно установить с помощью функции date_default_timezone_set
date_default_timezone_set('Europe/Kaliningrad');
// или через класс DateTimeImmutable с переданной таймзоной
$operationTime = new \DateTimeImmutable('now', new \DateTimeZone('Europe/Kaliningrad'))
```
### Аутентификация
#### С помощью ИНН и пароля
```php
use Shoman4eg\Nalog\ApiClient;

$apiClient = ApiClient::create();

// Если у вас есть accessToken то можете пропустить этот шаг
try {
    $accessToken = $apiClient->createNewAccessToken($username, $password);
} catch (\Shoman4eg\Nalog\Exception\Domain\UnauthorizedException $e) {
    var_dump($e->getMessage());
}

$apiClient->authenticate($accessToken);
```

#### По номеру телефона
Аутентификация по номеру телефона состоит из 2 шагов
Вам необходимо запросить авторизацию по телефону, временно сохранить возвращенный challengeToken, получить СМС с кодом подтверждения, а затем повторным запросом передать телефон, challengeToken и код подтверждения из СМС.

**Внимание:** имеется ограничение на отправку СМС с кодом подтверждения (одна СМС раз в 1-2 минуты).

#### 1. Получение СМС с кодом подтвержениея на ваш номер телефона и временного challengeToken'а:
```php
use Shoman4eg\Nalog\ApiClient;

try {
    $phoneChallengeResponse = ApiClient::createPhoneChallenge('79999999999');
    /**
     * $phoneChallengeResponse = [
     *  'challengeToken' => '00000000-0000-0000-0000-000000000000',
     *  'expireDate' => 2022-11-24T00:20:19.135436Z,
     *  'expireIn' => 120,
     *  ];
     */

} catch (\Shoman4eg\Nalog\Exception\Domain\UnauthorizedException $e) {
    var_dump($e->getMessage());
}
// Сохраните $phoneChallengeResponse['challengeToken']. Он вам потребуется на следующем шаге.

```
#### 2. Ввод номера телефона, challengeToken'а и СМС:
```php
use Shoman4eg\Nalog\ApiClient;

$apiClient = ApiClient::create();

try {
    $accessToken = $apiClient->createNewAccessTokenByPhone(
        '79999999999', // Номер телефона
        $phoneChallengeResponse['challengeToken'],
        '111111' // Код из СМС
    );
} catch (\Shoman4eg\Nalog\Exception\Domain\UnauthorizedException $e) {
    var_dump($e->getMessage());
}

$apiClient->authenticate($accessToken);
```

### Создать `income` c контрагентом по умолчанию
```php
$name = 'Предоставление информационных услуг #970/2495';
$amount = 1800.30;
$quantity = 1;
$operationTime = new DateTimeImmutable('2020-12-31 12:12:00');
$createdIncome = $apiClient->income()->create($name, $amount, $quantity, $operationTime);
```

### Создать `income` с несколькими позициями
```php
$name = 'Предоставление информационных услуг #970/2495';
$items = [
    new \Shoman4eg\Nalog\DTO\IncomeServiceItem($name, $amount = 1800.30, $quantity = 1),
    new \Shoman4eg\Nalog\DTO\IncomeServiceItem($name, $amount = 900, $quantity = 2),
    new \Shoman4eg\Nalog\DTO\IncomeServiceItem($name, $amount = '1399.99', $quantity = 3),
];
$operationTime = new DateTimeImmutable('2020-12-31 12:12:00');
$createdIncome = $apiClient->income()->createMultipleItems($items, $operationTime);
```

### Создать `income` различными контрагентами
```php
$name = 'Предоставление информационных услуг #970/2495';
$amount = 1800.30;
$quantity = 1;
$operationTime = new \DateTimeImmutable('2020-12-31 12:12:00');

$client = new \Shoman4eg\Nalog\DTO\IncomeClient(); // Default. All fields are empty IncomeType is FROM_INDIVIDUAL
// or
$client = new \Shoman4eg\Nalog\DTO\IncomeClient('+79009000000', 'Вася Пупкин', \Shoman4eg\Nalog\Enum\IncomeType::INDIVIDUAL, '390000000000');
// or
$client = new \Shoman4eg\Nalog\DTO\IncomeClient(null, 'Facebook Inc.', \Shoman4eg\Nalog\Enum\IncomeType::FOREIGN_AGENCY, '390000000000');
// or
$client = new \Shoman4eg\Nalog\DTO\IncomeClient(null, 'ИП Вася Пупкин Валерьевич', \Shoman4eg\Nalog\Enum\IncomeType::LEGAL_ENTITY, '7700000000');
$createdIncome = $apiClient->income()->create($name, $amount, $quantity, $operationTime, $client);
```

### Отменить `income`
```php
$receiptUuid = "20hykdxbp8"
$comment = \Shoman4eg\Nalog\Enum\CancelCommentType::CANCEL;
$partnerCode = null; // Default null
$operationTime = new \DateTimeImmutable('now'); //Default 'now'
$requestTime = new \DateTimeImmutable('now'); //Default 'now'
$incomeInfo = $apiClient->income()->cancel($receiptUuid, $comment, $operationTime, $requestTime, $partnerCode);
```

### Create Invoice
```php
// todo
```

### Cancel Invoice
```php
// todo
```

### Change payment type in Invoice
```php
// todo
```

### Получить информацию о текущем пользователе
```php
$userInfo = $apiClient->user()->get();
```

### Получить информацию о чеке
```php
// $receiptUuid = $createdincome->getApprovedReceiptUuid();

// Получить ссылку на который чек для печати
$receipt = $apiClient->receipt()->printUrl($receiptUuid);

// Данные по чеку в Json формате
$receipt = $apiClient->receipt()->json($receiptUuid);
```

## Использованные ресурсы
[Автоматизация для самозанятых: как интегрировать налог с IT проектом](https://habr.com/ru/post/436656/)

JS lib [alexstep/moy-nalog](https://github.com/alexstep/moy-nalog)

## Лог изменений
[Changelog](CHANGELOG.md): A complete changelog

## На кофе
If this project help you reduce time to develop, you can give me a cup of coffee :)

[Link to donate](https://www.tinkoff.ru/cf/7rZnC7N4bOO)

## License
The MIT License (MIT). Please see [License File](LICENSE) for more information.
