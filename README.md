# 商城系统源码支持多端（H5小程序APP支持）Thinkphp+Uniapp技术

源码获取：ym7k.com/11647/ 商城系统源码支持多端（H5小程序APP支持）Thinkphp+Uniapp技术


<img width="519" height="854" alt="2025020323033836" src="https://github.com/user-attachments/assets/21efad3c-813c-4411-b9de-5581b02d8d01" />
<img width="1919" height="933" alt="202502032304176" src="https://github.com/user-attachments/assets/d3ac0f12-155f-419c-86d7-492e87192c0f" />
ThinkPHP + UniApp 多端商城系统：赠送优惠券的工程架构与实现
引子：赠送优惠券不是“发一张券”那么简单
在商城系统的营销功能清单中，“赠送优惠券”看起来是最简单的一个。它不像拼团需要成团状态管理，不像秒杀需要高并发控制，不像分销需要层级追溯。它似乎只是“发一张券给用户”。

但真正实现过的人会知道，赠送优惠券的工程复杂度被严重低估了。它涉及四个核心问题：什么时候发（触发时机）、发给谁（目标用户）、发什么券（券的类型和规则）、怎么防止重复发（幂等性）。

触发时机可能是注册、下单、评价、分享、生日、节日——每种时机的业务逻辑不同。目标用户可能是新用户、老用户、特定会员等级、特定标签的用户——筛选逻辑不同。券的类型可能是满减券、折扣券、无门槛券——计算逻辑不同。幂等性要求在“同一用户在同一场景下只能领取一次”和“用户可以在不同场景领取多张”之间找到平衡。

当系统还要支持多端（H5、小程序、App）时，赠送优惠券的展示、领取、使用需要在三端保持一致。

本文以 ThinkPHP + UniApp 的多端商城系统为例，从优惠券的数据模型、赠送触发的场景设计、发放的并发控制、核销与过期处理，到多端一致性保证，展开这类系统的完整技术分析。

一、优惠券的数据模型
1.1 券模板表
优惠券的核心是券模板 + 用户券的两层结构。券模板定义了券的类型和规则，用户券是模板在用户身上的实例化。

sql
CREATE TABLE `coupon_template` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL COMMENT '券名称',
  `type` tinyint(1) NOT NULL COMMENT '1=满减券 2=折扣券 3=无门槛券',
  `value` decimal(10,2) NOT NULL COMMENT '面额或折扣',
  `min_amount` decimal(10,2) NOT NULL DEFAULT '0.00' COMMENT '使用门槛',
  `max_discount` decimal(10,2) DEFAULT NULL COMMENT '折扣券最高优惠',
  `total_count` int(11) NOT NULL DEFAULT '0' COMMENT '发行总量，0=不限',
  `issued_count` int(11) NOT NULL DEFAULT '0' COMMENT '已发放数量',
  `used_count` int(11) NOT NULL DEFAULT '0' COMMENT '已使用数量',
  `per_user_limit` int(11) NOT NULL DEFAULT '1' COMMENT '每人限领数量',
  `valid_type` tinyint(1) NOT NULL DEFAULT '1' COMMENT '1=固定日期 2=领取后N天',
  `valid_start` datetime DEFAULT NULL COMMENT '固定有效期开始',
  `valid_end` datetime DEFAULT NULL COMMENT '固定有效期结束',
  `valid_days` int(11) DEFAULT NULL COMMENT '领取后有效天数',
  `scope_type` tinyint(1) NOT NULL DEFAULT '0' COMMENT '0=全场 1=指定商品 2=指定分类',
  `scope_ids` text COMMENT '适用范围ID列表',
  `status` tinyint(1) NOT NULL DEFAULT '1',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `idx_status` (`status`),
  KEY `idx_valid` (`valid_start`, `valid_end`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
valid_type 字段是优惠券设计的核心。 固定日期（如“双 11 当天有效”）和领取后 N 天（如“领取后 7 天内有效”）是两种完全不同的有效期策略。固定日期适合大促活动，领取后 N 天适合日常赠送。

1.2 用户券表
sql
CREATE TABLE `user_coupon` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `template_id` int(11) NOT NULL,
  `coupon_code` varchar(32) NOT NULL COMMENT '券码',
  `status` tinyint(1) NOT NULL DEFAULT '0' COMMENT '0=未使用 1=已使用 2=已过期 3=已作废',
  `source` varchar(30) NOT NULL COMMENT '来源：register/order/activity/share/admin',
  `source_id` varchar(50) DEFAULT NULL COMMENT '来源ID（如订单号）',
  `valid_start` datetime NOT NULL,
  `valid_end` datetime NOT NULL,
  `used_at` datetime DEFAULT NULL,
  `order_id` int(11) DEFAULT NULL COMMENT '使用的订单',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_coupon_code` (`coupon_code`),
  KEY `idx_user_status` (`user_id`, `status`),
  KEY `idx_template` (`template_id`),
  KEY `idx_valid_end` (`valid_end`),
  KEY `idx_source` (`source`, `source_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
source 字段记录了券的来源。 这是防止重复发放的关键。当系统需要检查“用户是否已经领取过注册赠券”时，查询 source = 'register' AND user_id = X 即可。

coupon_code 是券的唯一标识。 它用于核销时的验证，也用于客服查询。

二、赠送触发的场景设计
2.1 六种赠送场景
赠送优惠券的触发场景可以归纳为六种。

注册赠送。 用户首次注册时，系统自动发放新人券。这是最常见的赠送场景。

下单赠送。 用户完成订单后，系统发放复购券，激励二次购买。

评价赠送。 用户完成商品评价后，系统发放评价奖励券。

分享赠送。 用户分享商品或活动到社交平台后，系统发放分享奖励券。

生日赠送。 用户生日当天，系统发放生日券。

管理员手动赠送。 运营人员针对特定用户或用户群体手动发放。

2.2 赠送服务的统一接口
六种场景共享同一套发放逻辑，但触发条件和参数不同。系统通过一个统一的赠送服务处理。

php
<?php
// app/common/service/CouponGrantService.php

namespace app\common\service;

use think\Db;
use think\facade\Log;

class CouponGrantService
{
    /**
     * 统一赠送接口
     */
    public function grant(int $userId, int $templateId, array $options = []): array
    {
        $template = Db::name('coupon_template')
            ->where('id', $templateId)
            ->where('status', 1)
            ->find();

        if (!$template) {
            throw new \Exception('优惠券不存在或已停用');
        }

        $source = $options['source'] ?? 'admin';
        $sourceId = $options['source_id'] ?? null;

        // 1. 检查库存
        if ($template['total_count'] > 0
            && $template['issued_count'] >= $template['total_count']) {
            throw new \Exception('优惠券已发完');
        }

        // 2. 检查每人限领
        if ($template['per_user_limit'] > 0) {
            $userCount = Db::name('user_coupon')
                ->where('user_id', $userId)
                ->where('template_id', $templateId)
                ->whereIn('status', [0, 1]) // 未使用或已使用
                ->count();

            if ($userCount >= $template['per_user_limit']) {
                throw new \Exception('您已领取过该优惠券');
            }
        }

        // 3. 检查同场景重复发放（幂等）
        if ($sourceId) {
            $exists = Db::name('user_coupon')
                ->where('user_id', $userId)
                ->where('template_id', $templateId)
                ->where('source', $source)
                ->where('source_id', $sourceId)
                ->count();

            if ($exists) {
                throw new \Exception('该场景已发放过优惠券');
            }
        }

        // 4. 计算有效期
        [$validStart, $validEnd] = $this->calculateValidPeriod($template);

        // 5. 发放
        Db::startTrans();
        try {
            $couponCode = $this->generateCouponCode();

            $couponId = Db::name('user_coupon')->insertGetId([
                'user_id' => $userId,
                'template_id' => $templateId,
                'coupon_code' => $couponCode,
                'status' => 0,
                'source' => $source,
                'source_id' => $sourceId,
                'valid_start' => $validStart,
                'valid_end' => $validEnd,
                'created_at' => date('Y-m-d H:i:s'),
            ]);

            // 更新发放数量
            Db::name('coupon_template')
                ->where('id', $templateId)
                ->inc('issued_count', 1)
                ->update();

            Db::commit();

            return [
                'coupon_id' => $couponId,
                'coupon_code' => $couponCode,
                'name' => $template['name'],
                'valid_end' => $validEnd,
            ];

        } catch (\Exception $e) {
            Db::rollback();
            Log::error('优惠券发放失败', [
                'user_id' => $userId,
                'template_id' => $templateId,
                'error' => $e->getMessage(),
            ]);
            throw $e;
        }
    }

    /**
     * 批量赠送（管理员手动赠送）
     */
    public function batchGrant(array $userIds, int $templateId, array $options = []): array
    {
        $success = 0;
        $failed = 0;
        $errors = [];

        foreach ($userIds as $userId) {
            try {
                $this->grant($userId, $templateId, $options);
                $success++;
            } catch (\Exception $e) {
                $failed++;
                $errors[] = "用户 {$userId}: " . $e->getMessage();
            }
        }

        return [
            'success' => $success,
            'failed' => $failed,
            'errors' => $errors,
        ];
    }

    /**
     * 计算有效期
     */
    protected function calculateValidPeriod(array $template): array
    {
        if ($template['valid_type'] == 1) {
            // 固定日期
            return [$template['valid_start'], $template['valid_end']];
        }

        // 领取后 N 天
        $start = date('Y-m-d H:i:s');
        $end = date('Y-m-d H:i:s', strtotime("+{$template['valid_days']} days"));
        return [$start, $end];
    }

    /**
     * 生成券码
     */
    protected function generateCouponCode(): string
    {
        do {
            $code = strtoupper(substr(md5(uniqid('', true)), 0, 12));
        } while (Db::name('user_coupon')->where('coupon_code', $code)->count());
        return $code;
    }
}
幂等性检查是核心设计。 通过 source + source_id 的组合，保证同一场景（如同一笔订单）不会重复发放优惠券。source_id 通常是订单号、活动 ID 或评价 ID。

2.3 场景触发的实现
每种赠送场景在对应的业务逻辑中调用统一接口。

php
<?php
// app/common/service/UserService.php

namespace app\common\service;

use think\Db;

class UserService
{
    /**
     * 用户注册
     */
    public function register(array $data): array
    {
        Db::startTrans();
        try {
            $userId = Db::name('user')->insertGetId([
                'phone' => $data['phone'],
                'nickname' => $data['nickname'] ?? '用户' . substr($data['phone'], -4),
                'created_at' => date('Y-m-d H:i:s'),
            ]);

            // 注册赠送新人券
            $this->grantRegisterCoupon($userId);

            Db::commit();

            return ['user_id' => $userId];

        } catch (\Exception $e) {
            Db::rollback();
            throw $e;
        }
    }

    /**
     * 注册赠券
     */
    protected function grantRegisterCoupon(int $userId): void
    {
        // 查找启用的注册赠券模板
        $template = Db::name('coupon_template')
            ->where('status', 1)
            ->where('grant_scene', 'register')
            ->find();

        if (!$template) return;

        try {
            $service = new CouponGrantService();
            $service->grant($userId, $template['id'], [
                'source' => 'register',
                'source_id' => (string)$userId,
            ]);
        } catch (\Exception $e) {
            // 赠券失败不影响注册流程
            \think\facade\Log::error('注册赠券失败', [
                'user_id' => $userId,
                'error' => $e->getMessage(),
            ]);
        }
    }
}
赠券失败不影响主流程。 用户注册成功后，即使赠券失败，注册本身也应该成功。这是容错设计的关键。

2.4 订单完成赠券
php
<?php
// app/common/service/OrderService.php

namespace app\common\service;

use think\Db;

class OrderService
{
    /**
     * 订单完成
     */
    public function complete(int $orderId): void
    {
        Db::startTrans();
        try {
            $order = Db::name('order')
                ->where('id', $orderId)
                ->lock(true)
                ->find();

            if ($order['status'] != 3) {
                throw new \Exception('订单状态不正确');
            }

            Db::name('order')
                ->where('id', $orderId)
                ->update([
                    'status' => 4,
                    'completed_at' => date('Y-m-d H:i:s'),
                ]);

            // 触发订单完成赠券
            $this->grantOrderCoupon($order);

            Db::commit();

        } catch (\Exception $e) {
            Db::rollback();
            throw $e;
        }
    }

    /**
     * 订单完成赠券
     */
    protected function grantOrderCoupon(array $order): void
    {
        $template = Db::name('coupon_template')
            ->where('status', 1)
            ->where('grant_scene', 'order_complete')
            ->find();

        if (!$template) return;

        try {
            $service = new CouponGrantService();
            $service->grant($order['user_id'], $template['id'], [
                'source' => 'order',
                'source_id' => $order['order_no'],
            ]);
        } catch (\Exception $e) {
            \think\facade\Log::error('订单完成赠券失败', [
                'order_no' => $order['order_no'],
                'error' => $e->getMessage(),
            ]);
        }
    }
}
source_id = order_no 保证了同一订单只赠一次券。 即使用户多次点击“确认收货”，也不会重复发放。

三、发放的并发控制
3.1 库存扣减的原子性
优惠券的库存扣减需要原子性。高并发下，多个用户同时领取同一张券时，可能出现超发。

php
// 使用 Redis 原子操作扣减库存
protected function deductStock(int $templateId): bool
{
    $key = "coupon:stock:{$templateId}";
    
    // 如果 Redis 中没有该键，从数据库加载
    if (!Redis::exists($key)) {
        $template = Db::name('coupon_template')->where('id', $templateId)->find();
        if ($template['total_count'] == 0) {
            return true; // 不限量
        }
        $remain = $template['total_count'] - $template['issued_count'];
        Redis::set($key, $remain);
    }

    // 原子扣减
    $script = <<<LUA
        local stock = redis.call('GET', KEYS[1])
        if not stock then return -1 end
        if tonumber(stock) <= 0 then return 0 end
        redis.call('DECR', KEYS[1])
        return 1
LUA;

    $result = Redis::eval($script, [$key], 1);
    return $result === 1;
}
Lua 脚本保证了“检查-扣减”的原子性。 这是防止超发的标准方案。

3.2 每人限领的并发控制
每人限领的检查也需要并发控制。如果两个请求同时检查“用户是否已领取”，都返回“未领取”，就会发放两张券。

php
// 使用 Redis 分布式锁
protected function checkPerUserLimit(int $userId, int $templateId, int $limit): bool
{
    if ($limit <= 0) return true;

    $lockKey = "coupon:lock:{$userId}:{$templateId}";
    $lockToken = uniqid('', true);

    // 获取锁
    $locked = Redis::set($lockKey, $lockToken, ['nx', 'ex' => 5]);
    if (!$locked) {
        throw new \Exception('操作过于频繁，请稍后再试');
    }

    try {
        $userCount = Db::name('user_coupon')
            ->where('user_id', $userId)
            ->where('template_id', $templateId)
            ->whereIn('status', [0, 1])
            ->count();

        return $userCount < $limit;

    } finally {
        // 释放锁
        $script = <<<LUA
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end
LUA;
        Redis::eval($script, [$lockKey, $lockToken], 1);
    }
}
分布式锁保证了“检查-发放”的原子性。 锁的过期时间设置为 5 秒，防止死锁。

四、核销与过期处理
4.1 使用优惠券
用户下单时，系统需要计算可用优惠券，并在订单创建时锁定券。

php
<?php
// app/common/service/CouponUseService.php

namespace app\common\service;

use think\Db;

class CouponUseService
{
    /**
     * 获取用户可用的优惠券
     */
    public function getAvailable(int $userId, float $orderAmount, array $items = []): array
    {
        $now = date('Y-m-d H:i:s');

        $coupons = Db::name('user_coupon')
            ->alias('uc')
            ->join('coupon_template ct', 'uc.template_id = ct.id')
            ->where('uc.user_id', $userId)
            ->where('uc.status', 0)
            ->where('uc.valid_start', '<=', $now)
            ->where('uc.valid_end', '>=', $now)
            ->field('uc.*, ct.name, ct.type, ct.value, ct.min_amount, ct.max_discount, ct.scope_type, ct.scope_ids')
            ->select()
            ->toArray();

        $available = [];
        foreach ($coupons as $coupon) {
            // 检查门槛
            if ($orderAmount < $coupon['min_amount']) continue;

            // 检查适用范围
            if (!$this->checkScope($coupon, $items)) continue;

            // 计算优惠金额
            $coupon['discount_amount'] = $this->calcDiscount($coupon, $orderAmount);
            $available[] = $coupon;
        }

        // 按优惠金额降序排序
        usort($available, function ($a, $b) {
            return $b['discount_amount'] <=> $a['discount_amount'];
        });

        return $available;
    }

    /**
     * 使用优惠券（下单时锁定）
     */
    public function use(int $userId, int $couponId, int $orderId): bool
    {
        Db::startTrans();
        try {
            $coupon = Db::name('user_coupon')
                ->where('id', $couponId)
                ->where('user_id', $userId)
                ->where('status', 0)
                ->lock(true)
                ->find();

            if (!$coupon) {
                throw new \Exception('优惠券不存在或已使用');
            }

            // 检查有效期
            $now = date('Y-m-d H:i:s');
            if ($now < $coupon['valid_start'] || $now > $coupon['valid_end']) {
                throw new \Exception('优惠券已过期');
            }

            // 锁定优惠券
            Db::name('user_coupon')
                ->where('id', $couponId)
                ->update([
                    'status' => 1,
                    'used_at' => date('Y-m-d H:i:s'),
                    'order_id' => $orderId,
                ]);

            // 更新模板使用数量
            Db::name('coupon_template')
                ->where('id', $coupon['template_id'])
                ->inc('used_count', 1)
                ->update();

            Db::commit();
            return true;

        } catch (\Exception $e) {
            Db::rollback();
            throw $e;
        }
    }

    /**
     * 计算优惠金额
     */
    protected function calcDiscount(array $coupon, float $orderAmount): float
    {
        switch ($coupon['type']) {
            case 1: // 满减券
                return min($coupon['value'], $orderAmount);

            case 2: // 折扣券
                $discount = $orderAmount * (1 - $coupon['value'] / 10);
                $maxDiscount = $coupon['max_discount'] ?? PHP_FLOAT_MAX;
                return round(min($discount, $maxDiscount), 2);

            case 3: // 无门槛券
                return min($coupon['value'], $orderAmount);

            default:
                return 0;
        }
    }

    /**
     * 检查适用范围
     */
    protected function checkScope(array $coupon, array $items): bool
    {
        if ($coupon['scope_type'] == 0) return true;

        $scopeIds = json_decode($coupon['scope_ids'] ?? '[]', true);
        if (empty($scopeIds)) return true;

        foreach ($items as $item) {
            if ($coupon['scope_type'] == 1) {
                // 指定商品
                if (in_array($item['product_id'], $scopeIds)) return true;
            } elseif ($coupon['scope_type'] == 2) {
                // 指定分类
                if (in_array($item['category_id'], $scopeIds)) return true;
            }
        }

        return false;
    }
}
锁定优惠券是下单时的关键操作。 用户下单后，优惠券状态变为“已使用”，防止同一张券被用于多个订单。

4.2 过期处理
优惠券过期需要定时任务批量处理。

php
<?php
// app/command/CouponExpire.php

namespace app\command;

use think\console\Command;
use think\console\Input;
use think\console\Output;
use think\facade\Db;

class CouponExpire extends Command
{
    protected function configure()
    {
        $this->setName('coupon:expire')
            ->setDescription('处理过期优惠券');
    }

    protected function execute(Input $input, Output $output)
    {
        $now = date('Y-m-d H:i:s');

        // 批量更新过期优惠券
        $affected = Db::name('user_coupon')
            ->where('status', 0)
            ->where('valid_end', '<', $now)
            ->update(['status' => 2]);

        $output->writeln("已处理过期优惠券：{$affected} 张");
    }
}
批量更新而非逐条更新。 一条 SQL 语句处理所有过期券，效率远高于循环处理。

4.3 过期提醒
在优惠券过期前 3 天，系统发送提醒通知。

php
<?php
// app/command/CouponRemind.php

namespace app\command;

use think\console\Command;
use think\console\Input;
use think\console\Output;
use think\facade\Db;
use think\facade\Queue;

class CouponRemind extends Command
{
    protected function configure()
    {
        $this->setName('coupon:remind')
            ->setDescription('发送优惠券过期提醒');
    }

    protected function execute(Input $input, Output $output)
    {
        $remindDate = date('Y-m-d H:i:s', strtotime('+3 days'));

        $coupons = Db::name('user_coupon')
            ->where('status', 0)
            ->where('valid_end', 'between', [
                date('Y-m-d H:i:s'),
                $remindDate,
            ])
            ->where('reminded', 0)
            ->select();

        foreach ($coupons as $coupon) {
            Queue::push('app\job\SendCouponRemind', [
                'coupon_id' => $coupon['id'],
                'user_id' => $coupon['user_id'],
            ], 'notification');

            Db::name('user_coupon')
                ->where('id', $coupon['id'])
                ->update(['reminded' => 1]);
        }

        $output->writeln("已发送提醒：" . count($coupons) . " 条");
    }
}
五、多端一致性保证
5.1 优惠券的后端统一计算
多端一致性的核心是所有优惠券的计算都在后端完成。前端只负责展示和交互，不参与金额计算。

php
<?php
// app/api/controller/CouponController.php

namespace app\api\controller;

use app\common\service\CouponUseService;

class CouponController
{
    /**
     * 获取用户可用优惠券
     */
    public function available()
    {
        $userId = $this->userId;
        $orderAmount = input('order_amount/f', 0);
        $items = input('items/a', []);

        $service = new CouponUseService();
        $coupons = $service->getAvailable($userId, $orderAmount, $items);

        return json([
            'code' => 200,
            'data' => $coupons,
        ]);
    }

    /**
     * 领取优惠券
     */
    public function receive()
    {
        $templateId = input('template_id/d');
        $userId = $this->userId;

        $service = new CouponGrantService();
        $result = $service->grant($userId, $templateId, [
            'source' => 'receive',
            'source_id' => (string)$templateId,
        ]);

        return json([
            'code' => 200,
            'data' => $result,
        ]);
    }
}
5.2 前端的一致性展示
前端在三端使用相同的组件展示优惠券。

vue
<!-- components/CouponList.vue -->
<template>
  <view class="coupon-list">
    <view
      v-for="coupon in coupons"
      :key="coupon.id"
      class="coupon-item"
      :class="{ disabled: coupon.disabled }"
    >
      <view class="coupon-left">
        <view class="value">
          <text v-if="coupon.type === 1" class="symbol">¥</text>
          <text class="number">{{ coupon.value }}</text>
          <text v-if="coupon.type === 2" class="symbol">折</text>
        </view>
        <text class="condition">
          {{ coupon.min_amount > 0 ? `满${coupon.min_amount}可用` : '无门槛' }}
        </text>
      </view>
      <view class="coupon-right">
        <text class="name">{{ coupon.name }}</text>
        <text class="valid">{{ formatValid(coupon) }}</text>
      </view>
      <view class="coupon-action">
        <text v-if="coupon.status === 0" @click="$emit('use', coupon)">使用</text>
        <text v-else-if="coupon.status === 2" class="expired">已过期</text>
      </view>
    </view>
  </view>
</template>

<script>
export default {
  props: {
    coupons: { type: Array, default: () => [] },
  },
  methods: {
    formatValid(coupon) {
      return `${coupon.valid_start.slice(0, 10)} 至 ${coupon.valid_end.slice(0, 10)}`;
    },
  },
};
</script>
展示逻辑在三端共享，只有交互细节（如点击事件）通过条件编译分化。

六、边界与张力
赠送优惠券在多端商城系统中，有几个需要清醒认识的边界。

第一，超发的风险。 如果库存扣减没有使用原子操作，高并发下可能超发。系统需要使用 Redis 原子操作或数据库乐观锁。

第二，重复发放的风险。 如果幂等检查不严格，同一场景可能重复发放。系统需要通过 source + source_id 的组合保证幂等。

第三，优惠券叠加的规则。 用户可能同时持有多张优惠券，系统需要明确“是否允许叠加使用”的规则。通常一张订单只能使用一张券。

第四，过期提醒的频率。 如果提醒太频繁，用户会反感；如果提醒太少，用户会错过。系统需要设置合理的提醒策略（如过期前 3 天提醒一次）。

第五，多端券包的一致性。 用户在小程序领取的券，在 App 中应该能看到。这要求券的数据存储在服务端，三端从同一数据源读取。

结语：赠送优惠券的工程本质是“场景化发放 + 幂等保证”
ThinkPHP + UniApp 多端商城系统的赠送优惠券，其工程本质可以归结为一句话：在不同的业务场景中，以幂等的方式向用户发放优惠券。
