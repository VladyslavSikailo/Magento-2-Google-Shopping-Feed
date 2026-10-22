# Custom shipping data and attributes

## How shipping works today

`RunAsRoot\GoogleShoppingFeed\DataProvider\AttributeHandlers\ShippingProvider` is the handler registered for the `shipping` attribute in `Enum\AttributesToImportEnumInterface::ATTRIBUTES`. It reads the core `shipping_tablerate` table (countries from `general/country/allow`, condition from the table rate config) and is the only built-in shipping source. Shops using other carriers (Colissimo, GLS, ...) get an empty shipping node.

For every product, `ProductToFeedAttributesRowMapper` calls `AttributeHandlerInterface::get(Product $product)` on the handler. The handler instance is created by `Factory\AttributeHandlerFactory` through the object manager, so a DI `<preference>` on the handler class is honored.

`get()` must return a list of entries, each with exactly the keys `country` and `price`:

```php
[
    ['country' => 'DE', 'price' => '12.00 EUR'],
    ['country' => 'FR', 'price' => '9.50 EUR'],
]
```

`Converter\ArrayToXmlConverter::getShipping()` renders each entry as:

```xml
<g:shipping>
    <g:country><![CDATA[DE]]></g:country>
    <g:price><![CDATA[12.00 EUR]]></g:price>
</g:shipping>
```

Notes:

- Both keys are read without checks, so always return both. Return `[]` for no shipping.
- `country` is a two-letter ISO code. `price` is used verbatim, so format it the way Google expects (`12.00 EUR`, amount, space, ISO currency code).
- The bundled `CurrencyAmountProvider` uses the store's currency formatting and can produce `EUR12.00` (no space) depending on locale. Building the string yourself with `sprintf('%.2F %s', $amount, $currencyCode)` is safer.
- The product's store is available via `$product->getStoreId()`.

## Plug in a different shipping source

Replace `ShippingProvider` with your own class using a preference. No change to the module is needed.

`app/code/Acme/FeedShipping/etc/module.xml` (load after the feed module):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Acme_FeedShipping">
        <sequence>
            <module name="RunAsRoot_GoogleShoppingFeed"/>
        </sequence>
    </module>
</config>
```

`registration.php`:

```php
<?php

declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(ComponentRegistrar::MODULE, 'Acme_FeedShipping', __DIR__);
```

`etc/di.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <preference for="RunAsRoot\GoogleShoppingFeed\DataProvider\AttributeHandlers\ShippingProvider"
                type="Acme\FeedShipping\DataProvider\CarrierShippingProvider"/>
</config>
```

`DataProvider/CarrierShippingProvider.php` (this example uses a flat rate for every allowed country; replace the body of `get()` with a lookup in your carrier's rate tables):

```php
<?php

declare(strict_types=1);

namespace Acme\FeedShipping\DataProvider;

use Magento\Catalog\Model\Product;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\NoSuchEntityException;
use RunAsRoot\GoogleShoppingFeed\ConfigProvider\AllowedCountriesProvider;
use RunAsRoot\GoogleShoppingFeed\DataProvider\AttributeHandlers\AttributeHandlerInterface;
use RunAsRoot\GoogleShoppingFeed\DataProvider\CurrencyProvider;

class CarrierShippingProvider implements AttributeHandlerInterface
{
    private const FLAT_PRICE = 12.0;

    public function __construct(
        private readonly AllowedCountriesProvider $allowedCountriesProvider,
        private readonly CurrencyProvider $currencyProvider
    ) {
    }

    /**
     * @return array<int, array{country: string, price: string}>
     * @throws LocalizedException
     * @throws NoSuchEntityException
     */
    public function get(Product $product): array
    {
        $storeId = (int)$product->getStoreId();
        $currencyCode = $this->currencyProvider->get($storeId)->getCode();
        $result = [];

        foreach ($this->allowedCountriesProvider->get($storeId) as $countryId) {
            $result[] = [
                'country' => $countryId,
                'price' => sprintf('%.2F %s', self::FLAT_PRICE, $currencyCode),
            ];
        }

        return $result;
    }
}
```

Then:

```bash
bin/magento module:enable Acme_FeedShipping
bin/magento setup:upgrade
bin/magento cache:flush
bin/magento run_as_root:product-feed:execute
```

The class must implement `AttributeHandlerInterface`, otherwise `AttributeHandlerFactory` throws `WrongInstanceException`. Use constructor injection to pull in your carrier module's rate repository or resource model.

The same preference mechanism works for any other handler class listed in `ATTRIBUTES`. Handlers such as `ColorProvider` or `EanProvider` are virtual types defined in `src/etc/di.xml`, so you can also point them at a different product attribute by redefining the `attributeCode` argument of that virtual type.

## Add an entirely new attribute

`ATTRIBUTES` is a class constant, so it cannot be extended through DI. Two steps are always required: register the attribute (so the handler runs) and render it (so it ends up in the XML).

1. Write a handler implementing `AttributeHandlerInterface` (or define a virtual type of `SimpleAttributeHandler` / `SelectAttributeHandler` with an `attributeCode` argument in `di.xml`, as the module does for `ean`, `color`, ...).
2. Register it. Either add an entry to `AttributesToImportEnumInterface::ATTRIBUTES` (when working on the module itself):

   ```php
   'my_attribute' => [
       AttributeConfigData::FIELD_NAME => 'my_attribute',
       AttributeConfigData::ATTRIBUTE_HANDLER => MyAttributeProvider::class,
   ],
   ```

   or, from an external module, add an `afterGet` plugin on `RunAsRoot\GoogleShoppingFeed\DataProvider\AttributesConfigListProvider::get()` that returns a new `AttributeConfigDataList` containing the original `getList()` items plus an extra `AttributeConfigData` (created via `AttributeConfigDataFactory` with `['data' => [FIELD_NAME => ..., ATTRIBUTE_HANDLER => ...]]`).
3. Render it. `ArrayToXmlConverter::getItem()` is private and lists every output tag explicitly. The handler's value is available in the row under its field name, but nothing is printed until the converter is changed. Options: edit `getItem()` in the module, or point a `<preference>` at your own subclass/copy of `ArrayToXmlConverter` that emits the extra tag. For simple optional text values you can add a line like `{$this->getOptionalProductAttr($row, 'my_attribute', 'google_tag_name')}` inside `getItem()`.

The shipping preference described above has been run against a Magento instance (preference resolved, handler output rendered by `ArrayToXmlConverter` as `<g:shipping>` nodes). The new-attribute plugin route is described from the code and was not executed.
