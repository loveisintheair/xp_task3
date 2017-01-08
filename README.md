1. Дубликация кода

В проекте существует множество команд, связанных с обработкой данных в БД.
Почти в каждой в методе execute вызывается один и тот же блок кода.

```php
class RedirectCommand extends ContainerAwareCommand
{
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $this->em = $this->getContainer()->get('doctrine')->getManager();
        $this->connection = $this->em->getConnection();
        $this->connection->getConfiguration()->setSQLLogger(null);

        // ...
    }
}
```

Рефакторинг: extract method + pull up method

```php
class DataProcessingCommand extends ContainerAwareCommand
{
    protected function initDatabase()
    {
        $this->em = $this->getContainer()->get('doctrine')->getManager();
        $this->connection = $this->em->getConnection();
        $this->connection->getConfiguration()->setSQLLogger(null);        
    }
}

class RedirectCommand extends DataProcessingCommand
{

}
```

2. Простые типы

Есть контроллер, который обращается к менеджеру сущностей для сохранения данных. При успешном сохранении возвращается id записи, при неуспешном - текст ошибки.

Рефакторинг: Создадим класс 
OperationResult 
- addError($error)
- setEntity($entity)
- array getErrors()
- bool getResult()
- entity getEntity()

В результате получим возможность получить статус, список ошибок и сущность из объекта предсказуемого типа.

3. Длинный метод

Есть адовый метод обновления кеша цен

```php
protected function updatePricesCache($prices, $nights = 1)
{
    $nights = (int) $nights;
    if ($nights < 1) {
        $nights = 1;
    }

    $hotelIds = [];
    $hotelPrices = [];

    foreach ($prices as $providerCode => $providerPrices) {
        if (empty($providerPrices['completeTime'])) {
            return;
        }

        foreach ($providerPrices['prices'] as $k => $price) {
            $hotelId = $price['id'];
            if (!empty($price['price'])) {
                $hotelIds[] = $hotelId;
                if (empty($hotelPrices[$hotelId]) || $hotelPrices[$hotelId] > $price['price']) {
                    $hotelPrices[$hotelId] = $price;
                }
            }
        }
    }

    $hotels = [];
    if (count($hotelIds)) {
        $hotels = $this->hotelRepository->findHotelPricesNeedToBeUpdated($hotelIds);
    }

    if (!count($hotels)) {
        return;
    }

    foreach ($hotels as $hotel) {
        $price = $hotelPrices[ $hotel->getId() ];
        $hotel->setPrice($price['price'] / $nights);

        if (!empty($price['oldPrice'])) {
            $hotel->setOldPrice($price['oldPrice'] / $nights);
        } else {
            $hotel->setOldPrice(0);
        }

        $hotel->setPriceUpdateTime(new \DateTime());

        $this->em->persist($hotel);
    }

    $this->em->flush();
}
```

Рефакторинг: выделение метода, возможно управление кешем цен можно и вовсе выделить в отдельный класс (?)

```php
protected function updatePricesCache($prices, $nights = 1)
{
    $nights = (int) $nights;
    if ($nights < 1) {
        $nights = 1;
    }

    list($hotelPrices, $hotelIds) = $this->getMinimalPrices($prices);

    $hotels = [];
    if (count($hotelIds)) {
        $hotels = $this->hotelRepository->findHotelPricesNeedToBeUpdated($hotelIds);
    }

    if (!count($hotels)) {
        return;
    }

    $this->persistHotelPrices();

    $this->em->flush();
}

protected function getMinimalPrices($prices)
{
    $hotelPrices = [];

    foreach ($prices as $providerCode => $providerPrices) {
        if (empty($providerPrices['completeTime'])) {
            return;
        }

        foreach ($providerPrices['prices'] as $k => $price) {
            $hotelId = $price['id'];
            if (!empty($price['price'])) {
                $hotelIds[] = $hotelId;
                if (empty($hotelPrices[$hotelId]) || $hotelPrices[$hotelId] > $price['price']) {
                    $hotelPrices[$hotelId] = $price;
                }
            }
        }
    }

    return [$hotelPrices, $hotelIds];
}

protected function persistHotelPrices($hotels, $hotelPrices)
{
    foreach ($hotels as $hotel) {
        $price = $hotelPrices[ $hotel->getId() ];
        $hotel->setPrice($price['price'] / $nights);

        if (!empty($price['oldPrice'])) {
            $hotel->setOldPrice($price['oldPrice'] / $nights);
        } else {
            $hotel->setOldPrice(0);
        }

        $hotel->setPriceUpdateTime(new \DateTime());

        $this->em->persist($hotel);
    }
}
```
