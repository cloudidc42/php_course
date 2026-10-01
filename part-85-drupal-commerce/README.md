# Part 85: Drupal Commerce 2
## ขั้นตอนที่ 2411-2440: Drupal Commerce Setup และ Customization

Drupal Commerce 2 เป็น e-commerce framework ที่ทรงพลังสำหรับ Drupal 8/9/10
ใช้ Symfony components และ Plugin system

---

## ขั้นตอนที่ 2411: ติดตั้ง Drupal Commerce

```bash
# ติดตั้ง Drupal ด้วย Composer
composer create-project drupal/recommended-project my_shop
cd my_shop

# ติดตั้ง Commerce
composer require drupal/commerce drupal/commerce_payment drupal/commerce_shipping

# Enable modules
drush en commerce commerce_product commerce_order commerce_payment commerce_checkout commerce_cart
```

---

## ขั้นตอนที่ 2412: Custom Product Type Module

```php
<?php
declare(strict_types=1);

// modules/custom/my_products/my_products.module

use Drupal\commerce_product\Entity\Product;
use Drupal\commerce_product\Entity\ProductVariation;

/**
 * Implements hook_ENTITY_TYPE_presave() for commerce_product.
 */
function my_products_commerce_product_presave(Product $product): void
{
    if ($product->bundle() === 'digital') {
        // Digital products ไม่ต้องการ shipping
        foreach ($product->getVariations() as $variation) {
            $variation->set('requires_shipping', FALSE);
            $variation->save();
        }
    }
}

/**
 * Implements hook_commerce_product_variation_type_info_alter().
 */
function my_products_commerce_product_variation_type_info_alter(array &$info): void
{
    // เพิ่ม trait ให้ custom variation types
    foreach ($info as $key => &$type) {
        if (isset($type['class']) && $type['class'] === ProductVariation::class) {
            $type['class'] = 'Drupal\my_products\Entity\CustomProductVariation';
        }
    }
}
```

---

## ขั้นตอนที่ 2413: Custom Price Resolver

```php
<?php
declare(strict_types=1);

namespace Drupal\my_products\Resolver;

use Drupal\commerce\Context;
use Drupal\commerce\PurchasableEntityInterface;
use Drupal\commerce_price\Price;
use Drupal\commerce_price\Resolver\PriceResolverInterface;

/**
 * Price resolver สำหรับ tier pricing
 */
class TierPriceResolver implements PriceResolverInterface
{
    /**
     * {@inheritdoc}
     */
    public function resolve(PurchasableEntityInterface $entity, string $quantity, Context $context): ?Price
    {
        if (!$entity->hasField('field_tier_pricing')) {
            return null;
        }

        $tier_pricing = $entity->get('field_tier_pricing')->getValue();
        if (empty($tier_pricing)) {
            return null;
        }

        $qty = (int) $quantity;
        $applicable_price = null;

        // เรียงตาม quantity จากมากไปน้อย
        usort($tier_pricing, fn($a, $b) => $b['min_quantity'] <=> $a['min_quantity']);

        foreach ($tier_pricing as $tier) {
            if ($qty >= (int)$tier['min_quantity']) {
                $applicable_price = new Price($tier['price'], 'THB');
                break;
            }
        }

        return $applicable_price;
    }
}
```

---

## ขั้นตอนที่ 2414: Custom Order Processor

```php
<?php
declare(strict_types=1);

namespace Drupal\my_products\OrderProcessor;

use Drupal\commerce_order\Adjustment;
use Drupal\commerce_order\Entity\OrderInterface;
use Drupal\commerce_order\OrderProcessorInterface;
use Drupal\commerce_price\Price;

/**
 * Order processor สำหรับ loyalty discount
 */
class LoyaltyDiscountProcessor implements OrderProcessorInterface
{
    /**
     * {@inheritdoc}
     */
    public function process(OrderInterface $order): void
    {
        $customer = $order->getCustomer();
        if ($customer->isAnonymous()) {
            return;
        }

        // คำนวณ loyalty tier ของ customer
        $total_spent = $this->getCustomerTotalSpent($customer->id());
        $discount_percentage = $this->getLoyaltyDiscount($total_spent);

        if ($discount_percentage <= 0) {
            return;
        }

        $subtotal = $order->getSubtotalPrice();
        if (!$subtotal) {
            return;
        }

        $discount_amount = $subtotal->multiply((string)($discount_percentage / 100));
        $discount = new Adjustment([
            'type' => 'custom',
            'label' => sprintf('Loyalty Discount (%d%%)', $discount_percentage),
            'amount' => $discount_amount->multiply('-1'),
            'percentage' => (string)($discount_percentage / 100),
            'source_id' => 'loyalty_discount',
            'included' => FALSE,
            'locked' => FALSE,
        ]);

        $order->addAdjustment($discount);
    }

    private function getCustomerTotalSpent(int $uid): float
    {
        $query = \Drupal::entityQuery('commerce_order')
            ->condition('uid', $uid)
            ->condition('state', 'completed')
            ->accessCheck(FALSE);

        $order_ids = $query->execute();
        $total = 0.0;

        foreach ($order_ids as $order_id) {
            $order = \Drupal\commerce_order\Entity\Order::load($order_id);
            if ($order) {
                $total += (float)$order->getTotalPrice()?->getNumber();
            }
        }

        return $total;
    }

    private function getLoyaltyDiscount(float $total_spent): int
    {
        return match(true) {
            $total_spent >= 100000 => 10, // 10% สำหรับ Platinum
            $total_spent >= 50000 => 7,   // 7% สำหรับ Gold
            $total_spent >= 10000 => 5,   // 5% สำหรับ Silver
            $total_spent >= 2000 => 2,    // 2% สำหรับ Bronze
            default => 0,
        };
    }
}
```

---

## ขั้นตอนที่ 2415: Custom Payment Gateway

```php
<?php
declare(strict_types=1);

namespace Drupal\my_payment\Plugin\Commerce\PaymentGateway;

use Drupal\commerce_payment\Entity\PaymentInterface;
use Drupal\commerce_payment\Entity\PaymentMethodInterface;
use Drupal\commerce_payment\Exception\PaymentGatewayException;
use Drupal\commerce_payment\Plugin\Commerce\PaymentGateway\OnsitePaymentGatewayBase;
use GuzzleHttp\Exception\RequestException;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * @CommercePaymentGateway(
 *   id = "kbank_payment",
 *   label = @Translation("KBank Payment"),
 *   display_label = @Translation("KBank"),
 *   forms = {
 *     "add-payment-method" = "Drupal\my_payment\PluginForm\KBankPaymentMethodAddForm",
 *   },
 *   payment_method_types = {"credit_card"},
 *   credit_card_types = {
 *     "visa", "mastercard", "jcb",
 *   },
 * )
 */
class KBankPayment extends OnsitePaymentGatewayBase
{
    /**
     * {@inheritdoc}
     */
    public function createPayment(PaymentInterface $payment, bool $capture = TRUE): void
    {
        $this->assertPaymentState($payment, ['new']);
        $payment_method = $payment->getPaymentMethod();
        $this->assertPaymentMethod($payment_method);

        $amount = $payment->getAmount();
        $order = $payment->getOrder();

        try {
            $response = $this->httpClient->post($this->configuration['api_url'] . '/charge', [
                'json' => [
                    'amount' => (int)($amount->getNumber() * 100),
                    'currency' => $amount->getCurrencyCode(),
                    'token' => $payment_method->getRemoteId(),
                    'description' => 'Order #' . $order->getOrderNumber(),
                    'metadata' => [
                        'order_id' => $order->id(),
                        'customer_email' => $order->getEmail(),
                    ],
                ],
                'headers' => [
                    'X-Api-Key' => $this->configuration['api_key'],
                ],
            ]);

            $data = json_decode($response->getBody(), true);

            $payment->setState($capture ? 'completed' : 'authorization');
            $payment->setRemoteId($data['transaction_id']);
            $payment->save();

        } catch (RequestException $e) {
            throw new PaymentGatewayException('Payment failed: ' . $e->getMessage(), $e->getCode(), $e);
        }
    }

    /**
     * {@inheritdoc}
     */
    public function refundPayment(PaymentInterface $payment, ?Price $amount = NULL): void
    {
        $this->assertPaymentState($payment, ['completed', 'partially_refunded']);
        $amount = $amount ?: $payment->getAmount();
        $this->assertRefundAmount($payment, $amount);

        try {
            $response = $this->httpClient->post(
                $this->configuration['api_url'] . '/refund/' . $payment->getRemoteId(),
                [
                    'json' => ['amount' => (int)($amount->getNumber() * 100)],
                    'headers' => ['X-Api-Key' => $this->configuration['api_key']],
                ]
            );

            $old_refunded_amount = $payment->getRefundedAmount();
            $new_refunded_amount = $old_refunded_amount->add($amount);

            if ($new_refunded_amount->lessThan($payment->getAmount())) {
                $payment->setState('partially_refunded');
            } else {
                $payment->setState('refunded');
            }

            $payment->setRefundedAmount($new_refunded_amount);
            $payment->save();

        } catch (RequestException $e) {
            throw new PaymentGatewayException('Refund failed: ' . $e->getMessage());
        }
    }
}
```

---

## ขั้นตอนที่ 2416: Cart และ Checkout Customization

```php
<?php
declare(strict_types=1);

namespace Drupal\my_shop\EventSubscriber;

use Drupal\commerce_cart\Event\CartEntityAddEvent;
use Drupal\commerce_cart\Event\CartEvents;
use Drupal\commerce_order\Entity\OrderInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class CartEventSubscriber implements EventSubscriberInterface
{
    /**
     * {@inheritdoc}
     */
    public static function getSubscribedEvents(): array
    {
        return [
            CartEvents::CART_ENTITY_ADD => 'onCartAdd',
            CartEvents::CART_ORDER_ITEM_UPDATE => 'onCartItemUpdate',
        ];
    }

    public function onCartAdd(CartEntityAddEvent $event): void
    {
        $order = $event->getCart();
        $order_item = $event->getOrderItem();

        // ตรวจสอบ stock ก่อนเพิ่มลง cart
        $product_variation = $order_item->getPurchasedEntity();
        if ($product_variation && $product_variation->hasField('field_stock')) {
            $stock = (int)$product_variation->get('field_stock')->value;
            $quantity = (int)$order_item->getQuantity();

            if ($quantity > $stock) {
                // ลด quantity ให้เท่ากับ stock
                $order_item->setQuantity($stock);
                $order_item->save();

                \Drupal::messenger()->addWarning(
                    t('สินค้ามีในสต็อกเพียง @stock ชิ้น', ['@stock' => $stock])
                );
            }
        }

        // เพิ่ม free gift เมื่อซื้อครบ 1000 บาท
        $this->checkFreeGift($order);
    }

    private function checkFreeGift(OrderInterface $order): void
    {
        $subtotal = $order->getSubtotalPrice();
        if (!$subtotal) return;

        $threshold = 100000; // 1000 บาท = 100000 สตางค์
        $free_gift_sku = 'FREE-GIFT-001';

        // ตรวจสอบว่ามี free gift อยู่แล้วหรือไม่
        $has_free_gift = FALSE;
        foreach ($order->getItems() as $item) {
            if ($item->getPurchasedEntity()?->getSku() === $free_gift_sku) {
                $has_free_gift = TRUE;
                break;
            }
        }

        if ((float)$subtotal->getNumber() >= $threshold && !$has_free_gift) {
            // เพิ่ม free gift โดยอัตโนมัติ
            \Drupal::messenger()->addMessage(
                t('ยินดีด้วย! คุณได้รับของแถมฟรี!')
            );
        }
    }
}
```

---

## ขั้นตอนที่ 2417: Order Management

```php
<?php
declare(strict_types=1);

namespace Drupal\my_shop\Service;

use Drupal\commerce_order\Entity\OrderInterface;
use Drupal\state_machine\Plugin\Workflow\WorkflowInterface;

class OrderManagementService
{
    public function getOrdersByStatus(string $state, int $limit = 50): array
    {
        $query = \Drupal::entityQuery('commerce_order')
            ->condition('state', $state)
            ->sort('placed', 'DESC')
            ->range(0, $limit)
            ->accessCheck(FALSE);

        $ids = $query->execute();
        return \Drupal\commerce_order\Entity\Order::loadMultiple($ids);
    }

    public function fulfillOrder(OrderInterface $order, string $tracking_number = ''): bool
    {
        if ($order->getState()->getId() !== 'processing') {
            return FALSE;
        }

        // เพิ่ม tracking number
        if ($tracking_number && $order->hasField('field_tracking_number')) {
            $order->set('field_tracking_number', $tracking_number);
        }

        // เปลี่ยน state เป็น fulfilled
        $transition = $order->getState()->getWorkflow()->getTransition('fulfill');
        if ($transition) {
            $order->getState()->applyTransition($transition);
        }

        $order->save();

        // ส่ง notification
        $this->sendShippingNotification($order, $tracking_number);

        return TRUE;
    }

    private function sendShippingNotification(OrderInterface $order, string $tracking): void
    {
        $customer_email = $order->getEmail();
        $mailManager = \Drupal::service('plugin.manager.mail');

        $mailManager->mail(
            'my_shop',
            'order_shipped',
            $customer_email,
            \Drupal::languageManager()->getDefaultLanguage()->getId(),
            [
                'order' => $order,
                'tracking_number' => $tracking,
            ]
        );
    }

    public function getOrderStats(): array
    {
        $states = ['pending', 'processing', 'completed', 'canceled'];
        $stats = [];

        foreach ($states as $state) {
            $count = \Drupal::entityQuery('commerce_order')
                ->condition('state', $state)
                ->condition('placed', strtotime('-30 days'), '>=')
                ->accessCheck(FALSE)
                ->count()
                ->execute();

            $stats[$state] = (int)$count;
        }

        return $stats;
    }
}
```

---

## สรุปบทที่ 85

| หัวข้อ | Component | ประโยชน์ |
|--------|----------|---------|
| Price Resolver | PriceResolverInterface | Tier pricing |
| Order Processor | OrderProcessorInterface | Loyalty discount |
| Payment Gateway | OnsitePaymentGatewayBase | Custom payment |
| Cart Events | CartEventSubscriber | Stock check |
| Order Management | Service layer | Fulfillment |
| Workflow | State machine | Order states |

ถัดไป → Part 86: Microservices Patterns
