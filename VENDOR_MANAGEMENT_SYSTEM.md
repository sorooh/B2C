# 🎯 نظام إدارة البائعين المتقدم - Vendor Management System
## اختيار الأسواق + المنافسة على المنتجات + مراقبة الجودة

---

## 📋 جدول المحتويات
1. [اختيار الأسواق للبائع](#1-اختيار-الأسواق-للبائع)
2. [آلية المنافسة على المنتج الواحد](#2-آلية-المنافسة-على-المنتج-الواحد)
3. [نظام مراقبة جودة البائعين](#3-نظام-مراقبة-جودة-البائعين)
4. [التكامل التقني](#4-التكامل-التقني)

---

## 1. اختيار الأسواق للبائع

### 1.1 الفكرة الأساسية
كل بائع يختار **الأسواق الخارجية** التي يريد البيع عليها (Amazon, Bol.com, eBay, إلخ)

### 1.2 الهيكل التقني

```python
# models.py - نموذج إدارة الأسواق

class VendorMarketplacePreference(models.Model):
    """تفضيلات البائع للأسواق المختلفة"""
    
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE)
    marketplace = models.ForeignKey(MarketplaceConfig, on_delete=models.CASCADE)
    
    # Enable/Disable
    is_enabled = models.BooleanField(default=False)
    
    # Authentication Credentials
    api_credentials = models.JSONField(default=dict)
    # Example: {"client_id": "xxx", "client_secret": "xxx", "seller_id": "xxx"}
    
    # Auto-sync Settings
    auto_sync_products = models.BooleanField(default=True)
    auto_sync_inventory = models.BooleanField(default=True)
    auto_sync_prices = models.BooleanField(default=True)
    sync_interval = models.IntegerField(default=3600)  # seconds
    
    # Product Selection Strategy
    SYNC_STRATEGY_CHOICES = [
        ('all', 'جميع المنتجات'),
        ('selected', 'منتجات محددة'),
        ('category', 'حسب الفئة'),
        ('margin', 'حسب هامش الربح'),
    ]
    sync_strategy = models.CharField(max_length=20, choices=SYNC_STRATEGY_CHOICES, default='all')
    
    # Pricing Strategy
    PRICING_STRATEGY_CHOICES = [
        ('same', 'نفس السعر'),
        ('percentage', 'إضافة نسبة مئوية'),
        ('fixed', 'إضافة مبلغ ثابت'),
        ('competitive', 'تسعير تنافسي تلقائي'),
    ]
    pricing_strategy = models.CharField(max_length=20, choices=PRICING_STRATEGY_CHOICES, default='same')
    pricing_adjustment = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    # Example: pricing_strategy='percentage', pricing_adjustment=10.00 → السعر + 10%
    
    # Commission & Fees Handling
    include_commission_in_price = models.BooleanField(default=True)
    # إذا True: السعر النهائي يشمل عمولة السوق
    
    # Performance Tracking
    total_products_synced = models.IntegerField(default=0)
    active_listings = models.IntegerField(default=0)
    total_sales = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    total_orders = models.IntegerField(default=0)
    
    # Status
    last_sync = models.DateTimeField(null=True, blank=True)
    sync_errors_count = models.IntegerField(default=0)
    is_active = models.BooleanField(default=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        unique_together = ['vendor', 'marketplace']
        verbose_name = 'تفضيلات الأسواق'
        verbose_name_plural = 'تفضيلات الأسواق'

class VendorProductMarketplaceMapping(models.Model):
    """ربط المنتجات بالأسواق - للتحكم الدقيق"""
    
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE)
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    marketplace_preference = models.ForeignKey(VendorMarketplacePreference, on_delete=models.CASCADE)
    
    # Override Pricing for this specific product+marketplace
    custom_price = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    custom_title = models.CharField(max_length=500, blank=True)
    custom_description = models.TextField(blank=True)
    
    # Status
    is_listed = models.BooleanField(default=False)
    marketplace_product_id = models.CharField(max_length=100, blank=True)
    
    # Performance
    views = models.IntegerField(default=0)
    sales = models.IntegerField(default=0)
    revenue = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    
    last_synced = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        unique_together = ['product', 'marketplace_preference']
```

### 1.3 واجهة المستخدم (Vendor Dashboard)

```typescript
// components/vendor/MarketplaceSettings.tsx

interface MarketplaceSettingsProps {
  vendorId: string;
}

export default function MarketplaceSettings({ vendorId }: MarketplaceSettingsProps) {
  const [marketplaces, setMarketplaces] = useState<Marketplace[]>([]);
  const [preferences, setPreferences] = useState<VendorPreference[]>([]);
  
  return (
    <div className="space-y-6">
      <h2 className="text-2xl font-bold">إعدادات الأسواق الخارجية</h2>
      
      {/* قائمة الأسواق المتاحة */}
      <div className="grid grid-cols-2 gap-4">
        {marketplaces.map((marketplace) => (
          <MarketplaceCard
            key={marketplace.id}
            marketplace={marketplace}
            preference={preferences.find(p => p.marketplace_id === marketplace.id)}
            onToggle={handleToggleMarketplace}
            onConfigure={handleConfigureMarketplace}
          />
        ))}
      </div>
    </div>
  );
}

function MarketplaceCard({ marketplace, preference, onToggle, onConfigure }) {
  return (
    <Card className={preference?.is_enabled ? 'border-green-500' : 'border-gray-200'}>
      <CardHeader>
        <div className="flex justify-between items-center">
          <div className="flex items-center gap-3">
            <img src={marketplace.logo} alt={marketplace.name} className="w-12 h-12" />
            <div>
              <h3 className="font-semibold">{marketplace.display_name}</h3>
              <p className="text-sm text-gray-500">عمولة: {marketplace.commission_rate}%</p>
            </div>
          </div>
          <Switch
            checked={preference?.is_enabled || false}
            onCheckedChange={() => onToggle(marketplace.id)}
          />
        </div>
      </CardHeader>
      
      {preference?.is_enabled && (
        <CardContent>
          <div className="space-y-4">
            {/* استراتيجية المزامنة */}
            <div>
              <Label>استراتيجية المنتجات</Label>
              <Select value={preference.sync_strategy}>
                <SelectTrigger>
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="all">جميع المنتجات</SelectItem>
                  <SelectItem value="selected">منتجات محددة</SelectItem>
                  <SelectItem value="category">حسب الفئة</SelectItem>
                  <SelectItem value="margin">حسب هامش الربح</SelectItem>
                </SelectContent>
              </Select>
            </div>
            
            {/* استراتيجية التسعير */}
            <div>
              <Label>استراتيجية التسعير</Label>
              <div className="flex gap-2">
                <Select value={preference.pricing_strategy}>
                  <SelectTrigger className="flex-1">
                    <SelectValue />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="same">نفس السعر</SelectItem>
                    <SelectItem value="percentage">إضافة نسبة %</SelectItem>
                    <SelectItem value="fixed">إضافة مبلغ ثابت</SelectItem>
                    <SelectItem value="competitive">تسعير تنافسي</SelectItem>
                  </SelectContent>
                </Select>
                
                {preference.pricing_strategy !== 'same' && (
                  <Input
                    type="number"
                    value={preference.pricing_adjustment}
                    className="w-24"
                    placeholder="0.00"
                  />
                )}
              </div>
            </div>
            
            {/* إعدادات المزامنة */}
            <div className="space-y-2">
              <div className="flex items-center justify-between">
                <Label>مزامنة المخزون تلقائياً</Label>
                <Switch checked={preference.auto_sync_inventory} />
              </div>
              <div className="flex items-center justify-between">
                <Label>مزامنة الأسعار تلقائياً</Label>
                <Switch checked={preference.auto_sync_prices} />
              </div>
            </div>
            
            {/* الإحصائيات */}
            <div className="pt-4 border-t">
              <div className="grid grid-cols-3 gap-4 text-center">
                <div>
                  <p className="text-sm text-gray-500">المنتجات المفعلة</p>
                  <p className="text-xl font-bold">{preference.active_listings}</p>
                </div>
                <div>
                  <p className="text-sm text-gray-500">الطلبات</p>
                  <p className="text-xl font-bold">{preference.total_orders}</p>
                </div>
                <div>
                  <p className="text-sm text-gray-500">المبيعات</p>
                  <p className="text-xl font-bold">€{preference.total_sales}</p>
                </div>
              </div>
            </div>
            
            <Button onClick={() => onConfigure(marketplace.id)} className="w-full">
              إعدادات متقدمة
            </Button>
          </div>
        </CardContent>
      )}
    </Card>
  );
}
```

### 1.4 APIs للإدارة

```python
# views.py - APIs

from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response

class VendorMarketplaceViewSet(viewsets.ModelViewSet):
    """إدارة تفضيلات الأسواق للبائع"""
    
    @action(detail=False, methods=['get'])
    def available_marketplaces(self, request):
        """قائمة الأسواق المتاحة"""
        marketplaces = MarketplaceConfig.objects.filter(is_active=True)
        vendor = request.user.vendor_profile
        
        # جلب تفضيلات البائع الحالية
        preferences = VendorMarketplacePreference.objects.filter(vendor=vendor)
        preferences_dict = {p.marketplace_id: p for p in preferences}
        
        data = []
        for marketplace in marketplaces:
            data.append({
                'id': marketplace.id,
                'name': marketplace.name,
                'display_name': marketplace.display_name,
                'commission_rate': float(marketplace.commission_rate),
                'logo': marketplace.logo_url,
                'is_enabled': preferences_dict.get(marketplace.id, {}).is_enabled if marketplace.id in preferences_dict else False,
                'preference': MarketplacePreferenceSerializer(preferences_dict.get(marketplace.id)).data if marketplace.id in preferences_dict else None
            })
        
        return Response(data)
    
    @action(detail=True, methods=['post'])
    def toggle_marketplace(self, request, pk=None):
        """تفعيل/تعطيل سوق معين"""
        marketplace = get_object_or_404(MarketplaceConfig, pk=pk)
        vendor = request.user.vendor_profile
        
        preference, created = VendorMarketplacePreference.objects.get_or_create(
            vendor=vendor,
            marketplace=marketplace
        )
        
        preference.is_enabled = not preference.is_enabled
        preference.save()
        
        return Response({
            'status': 'success',
            'is_enabled': preference.is_enabled
        })
    
    @action(detail=True, methods=['put'])
    def update_settings(self, request, pk=None):
        """تحديث إعدادات سوق معين"""
        marketplace = get_object_or_404(MarketplaceConfig, pk=pk)
        vendor = request.user.vendor_profile
        
        preference = get_object_or_404(
            VendorMarketplacePreference,
            vendor=vendor,
            marketplace=marketplace
        )
        
        # تحديث الإعدادات
        serializer = MarketplacePreferenceSerializer(
            preference, 
            data=request.data, 
            partial=True
        )
        
        if serializer.is_valid():
            serializer.save()
            
            # إطلاق مهمة مزامنة في الخلفية
            if preference.is_enabled and preference.auto_sync_products:
                sync_vendor_products_to_marketplace.delay(
                    vendor.id, 
                    marketplace.id
                )
            
            return Response(serializer.data)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    @action(detail=True, methods=['post'])
    def sync_products(self, request, pk=None):
        """مزامنة يدوية للمنتجات"""
        marketplace = get_object_or_404(MarketplaceConfig, pk=pk)
        vendor = request.user.vendor_profile
        
        # إطلاق مهمة المزامنة
        task = sync_vendor_products_to_marketplace.delay(vendor.id, marketplace.id)
        
        return Response({
            'status': 'sync_started',
            'task_id': task.id,
            'message': 'تم بدء عملية المزامنة'
        })
```

---

## 2. آلية المنافسة على المنتج الواحد

### 2.1 المشكلة
عندما يكون هناك **أكثر من بائع** يبيع نفس المنتج → من يحصل على البيع؟

### 2.2 نظام "Buy Box" (صندوق الشراء)

```python
# models.py - نظام المنافسة

class ProductCompetition(models.Model):
    """إدارة المنافسة على المنتج الواحد"""
    
    product = models.OneToOneField(Product, on_delete=models.CASCADE, related_name='competition')
    
    # عدد البائعين المتنافسين
    total_vendors = models.IntegerField(default=0)
    
    # البائع الفائز الحالي (الذي يظهر في Buy Box)
    current_winner = models.ForeignKey(
        VendorProfile, 
        on_delete=models.SET_NULL, 
        null=True, 
        related_name='won_products'
    )
    
    # آخر حساب للفائز
    last_calculated = models.DateTimeField(auto_now=True)
    
    # تكرار إعادة الحساب
    recalculation_interval = models.IntegerField(default=3600)  # seconds
    
    class Meta:
        verbose_name = 'منافسة المنتج'
        verbose_name_plural = 'منافسات المنتجات'

class VendorProductOffer(models.Model):
    """عرض البائع للمنتج"""
    
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='vendor_offers')
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE)
    
    # السعر والتوفر
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock_quantity = models.IntegerField(default=0)
    is_available = models.BooleanField(default=True)
    
    # معلومات الشحن
    SHIPPING_SPEED_CHOICES = [
        ('same_day', 'نفس اليوم'),
        ('next_day', 'اليوم التالي'),
        ('2-3_days', '2-3 أيام'),
        ('3-5_days', '3-5 أيام'),
        ('1_week', 'أسبوع'),
    ]
    shipping_speed = models.CharField(max_length=20, choices=SHIPPING_SPEED_CHOICES, default='2-3_days')
    shipping_cost = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    free_shipping_threshold = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    # Buy Box Score (يحسب تلقائياً)
    buy_box_score = models.FloatField(default=0)
    
    # إحصائيات الأداء
    total_sales = models.IntegerField(default=0)
    conversion_rate = models.FloatField(default=0)  # نسبة التحويل
    
    # الحالة
    is_active = models.BooleanField(default=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        unique_together = ['product', 'vendor']
        ordering = ['-buy_box_score']

class BuyBoxHistory(models.Model):
    """تاريخ الفائزين بـ Buy Box"""
    
    product = models.ForeignKey(Product, on_delete=models.CASCADE)
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE)
    
    won_at = models.DateTimeField(auto_now_add=True)
    lost_at = models.DateTimeField(null=True, blank=True)
    
    # الإحصائيات خلال الفترة
    impressions = models.IntegerField(default=0)  # مرات الظهور
    clicks = models.IntegerField(default=0)  # النقرات
    conversions = models.IntegerField(default=0)  # المبيعات
    revenue = models.DecimalField(max_digits=12, decimal_places=2, default=0)
```

### 2.3 خوارزمية حساب Buy Box Score

```python
# services/buy_box_calculator.py

from decimal import Decimal
from typing import Dict

class BuyBoxCalculator:
    """حساب نقاط Buy Box لكل بائع"""
    
    # الأوزان (يمكن تعديلها حسب الاستراتيجية)
    WEIGHTS = {
        'price': 0.30,           # 30% - السعر
        'vendor_rating': 0.25,   # 25% - تقييم البائع
        'shipping_speed': 0.20,  # 20% - سرعة الشحن
        'shipping_cost': 0.10,   # 10% - تكلفة الشحن
        'stock_level': 0.05,     # 5% - مستوى المخزون
        'conversion_rate': 0.10, # 10% - معدل التحويل
    }
    
    @classmethod
    def calculate_score(cls, offer: VendorProductOffer, competitors: list) -> float:
        """حساب النقاط للعرض مقارنة بالمنافسين"""
        
        scores = {}
        
        # 1. نقاط السعر (أقل سعر = أعلى نقاط)
        prices = [o.price for o in competitors]
        min_price = min(prices)
        max_price = max(prices)
        
        if max_price > min_price:
            price_score = 1 - ((offer.price - min_price) / (max_price - min_price))
        else:
            price_score = 1.0
        scores['price'] = price_score * cls.WEIGHTS['price']
        
        # 2. نقاط تقييم البائع
        vendor_rating = offer.vendor.rating or 0
        scores['vendor_rating'] = (vendor_rating / 5.0) * cls.WEIGHTS['vendor_rating']
        
        # 3. نقاط سرعة الشحن
        shipping_speed_scores = {
            'same_day': 1.0,
            'next_day': 0.9,
            '2-3_days': 0.7,
            '3-5_days': 0.5,
            '1_week': 0.3,
        }
        scores['shipping_speed'] = shipping_speed_scores.get(offer.shipping_speed, 0.5) * cls.WEIGHTS['shipping_speed']
        
        # 4. نقاط تكلفة الشحن (شحن مجاني = أعلى نقاط)
        if offer.shipping_cost == 0:
            shipping_cost_score = 1.0
        else:
            # كلما أقل التكلفة، أعلى النقاط
            max_shipping = 10.0  # افتراض أقصى تكلفة
            shipping_cost_score = max(0, 1 - (float(offer.shipping_cost) / max_shipping))
        scores['shipping_cost'] = shipping_cost_score * cls.WEIGHTS['shipping_cost']
        
        # 5. نقاط المخزون (مخزون كافٍ = نقاط أعلى)
        if offer.stock_quantity >= 100:
            stock_score = 1.0
        elif offer.stock_quantity >= 50:
            stock_score = 0.8
        elif offer.stock_quantity >= 10:
            stock_score = 0.6
        elif offer.stock_quantity > 0:
            stock_score = 0.3
        else:
            stock_score = 0
        scores['stock_level'] = stock_score * cls.WEIGHTS['stock_level']
        
        # 6. نقاط معدل التحويل
        conversion_score = min(offer.conversion_rate / 10.0, 1.0)  # نسبة التحويل / 10%
        scores['conversion_rate'] = conversion_score * cls.WEIGHTS['conversion_rate']
        
        # المجموع الكلي
        total_score = sum(scores.values())
        
        return round(total_score, 4)
    
    @classmethod
    def determine_winner(cls, product_id: int) -> VendorProfile:
        """تحديد الفائز بـ Buy Box"""
        
        # جلب جميع العروض النشطة للمنتج
        offers = VendorProductOffer.objects.filter(
            product_id=product_id,
            is_active=True,
            is_available=True,
            stock_quantity__gt=0
        ).select_related('vendor')
        
        if not offers.exists():
            return None
        
        # حساب النقاط لكل عرض
        for offer in offers:
            competitors = list(offers)
            offer.buy_box_score = cls.calculate_score(offer, competitors)
            offer.save(update_fields=['buy_box_score'])
        
        # الفائز = أعلى نقاط
        winner_offer = offers.order_by('-buy_box_score').first()
        
        # تحديث ProductCompetition
        competition, _ = ProductCompetition.objects.get_or_create(
            product_id=product_id
        )
        
        # حفظ في التاريخ إذا تغير الفائز
        if competition.current_winner != winner_offer.vendor:
            # إنهاء الفترة السابقة
            if competition.current_winner:
                BuyBoxHistory.objects.filter(
                    product_id=product_id,
                    vendor=competition.current_winner,
                    lost_at__isnull=True
                ).update(lost_at=timezone.now())
            
            # بدء فترة جديدة
            BuyBoxHistory.objects.create(
                product_id=product_id,
                vendor=winner_offer.vendor
            )
            
            competition.current_winner = winner_offer.vendor
        
        competition.total_vendors = offers.count()
        competition.save()
        
        return winner_offer.vendor
```

### 2.4 Celery Task للحساب التلقائي

```python
# tasks.py

from celery import shared_task
from .services.buy_box_calculator import BuyBoxCalculator

@shared_task
def recalculate_buy_box_winners():
    """إعادة حساب الفائزين بـ Buy Box لجميع المنتجات التنافسية"""
    
    # المنتجات التي لديها أكثر من بائع
    competitions = ProductCompetition.objects.filter(total_vendors__gt=1)
    
    for competition in competitions:
        try:
            BuyBoxCalculator.determine_winner(competition.product_id)
        except Exception as e:
            logger.error(f"Error calculating Buy Box for product {competition.product_id}: {e}")
    
    return f"Recalculated Buy Box for {competitions.count()} products"

@shared_task
def update_vendor_conversion_rates():
    """تحديث معدلات التحويل للبائعين"""
    
    offers = VendorProductOffer.objects.filter(is_active=True)
    
    for offer in offers:
        # حساب معدل التحويل من آخر 30 يوم
        thirty_days_ago = timezone.now() - timedelta(days=30)
        
        history = BuyBoxHistory.objects.filter(
            product=offer.product,
            vendor=offer.vendor,
            won_at__gte=thirty_days_ago
        ).first()
        
        if history and history.impressions > 0:
            offer.conversion_rate = (history.conversions / history.impressions) * 100
            offer.save(update_fields=['conversion_rate'])
```

### 2.5 واجهة العرض للعميل

```typescript
// components/product/BuyBox.tsx

interface BuyBoxProps {
  product: Product;
  offers: VendorOffer[];
}

export default function BuyBox({ product, offers }: BuyBoxProps) {
  const winningOffer = offers[0]; // الفائز = أول عرض (أعلى score)
  const otherOffers = offers.slice(1);
  
  return (
    <div className="space-y-4">
      {/* العرض الفائز - Buy Box */}
      <Card className="border-2 border-green-500">
        <CardHeader>
          <Badge className="w-fit bg-green-500">✓ العرض الأفضل</Badge>
        </CardHeader>
        <CardContent className="space-y-4">
          <div className="flex justify-between items-center">
            <div>
              <p className="text-3xl font-bold">€{winningOffer.price}</p>
              {winningOffer.shipping_cost === 0 && (
                <p className="text-sm text-green-600">شحن مجاني</p>
              )}
            </div>
            <Button size="lg" className="px-8">
              أضف للسلة
            </Button>
          </div>
          
          <div className="flex items-center gap-2">
            <Avatar src={winningOffer.vendor.logo} />
            <div>
              <p className="font-medium">{winningOffer.vendor.name}</p>
              <div className="flex items-center gap-1">
                <StarRating rating={winningOffer.vendor.rating} />
                <span className="text-sm text-gray-500">
                  ({winningOffer.vendor.reviews_count})
                </span>
              </div>
            </div>
          </div>
          
          <div className="flex gap-2 text-sm text-gray-600">
            <ShippingIcon />
            <span>توصيل: {getShippingLabel(winningOffer.shipping_speed)}</span>
          </div>
          
          <div className="flex gap-2 text-sm text-gray-600">
            <StockIcon />
            <span className={winningOffer.stock_quantity > 10 ? 'text-green-600' : 'text-orange-600'}>
              {winningOffer.stock_quantity > 10 ? 'متوفر بكميات كبيرة' : `${winningOffer.stock_quantity} قطع متبقية`}
            </span>
          </div>
        </CardContent>
      </Card>
      
      {/* العروض الأخرى */}
      {otherOffers.length > 0 && (
        <Accordion type="single" collapsible>
          <AccordionItem value="other-offers">
            <AccordionTrigger>
              <span className="text-sm text-gray-600">
                {otherOffers.length} عرض إضافي من بائعين آخرين
              </span>
            </AccordionTrigger>
            <AccordionContent>
              <div className="space-y-2">
                {otherOffers.map((offer) => (
                  <Card key={offer.id} className="p-4">
                    <div className="flex justify-between items-center">
                      <div className="flex items-center gap-3">
                        <Avatar src={offer.vendor.logo} size="sm" />
                        <div>
                          <p className="font-medium">{offer.vendor.name}</p>
                          <StarRating rating={offer.vendor.rating} size="sm" />
                        </div>
                      </div>
                      
                      <div className="text-right">
                        <p className="text-xl font-bold">€{offer.price}</p>
                        <p className="text-xs text-gray-500">
                          + €{offer.shipping_cost} شحن
                        </p>
                      </div>
                      
                      <Button variant="outline" size="sm">
                        اختر
                      </Button>
                    </div>
                  </Card>
                ))}
              </div>
            </AccordionContent>
          </AccordionItem>
        </Accordion>
      )}
    </div>
  );
}
```

---

## 3. نظام مراقبة جودة البائعين

### 3.1 نظام Performance Score (مثل Bol.com)

```python
# models.py - نظام مراقبة الجودة

class VendorPerformanceScore(models.Model):
    """نقاط أداء البائع (0-100)"""
    
    vendor = models.OneToOneField(VendorProfile, on_delete=models.CASCADE, related_name='performance')
    
    # النقاط الإجمالية (0-100)
    overall_score = models.FloatField(default=0)
    
    # مكونات النقاط
    delivery_score = models.FloatField(default=0)      # سرعة التسليم
    cancellation_score = models.FloatField(default=0)  # نسبة الإلغاء
    customer_service_score = models.FloatField(default=0)  # خدمة العملاء
    product_quality_score = models.FloatField(default=0)   # جودة المنتجات
    return_rate_score = models.FloatField(default=0)       # نسبة الإرجاع
    
    # الحالة
    STATUS_CHOICES = [
        ('excellent', '⭐ ممتاز (90-100)'),
        ('good', '👍 جيد (70-89)'),
        ('average', '😐 متوسط (50-69)'),
        ('poor', '⚠️ ضعيف (30-49)'),
        ('critical', '🚨 حرج (<30)'),
    ]
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='average')
    
    # آخر تحديث
    last_calculated = models.DateTimeField(auto_now=True)
    
    class Meta:
        verbose_name = 'أداء البائع'
        verbose_name_plural = 'أداء البائعين'

class VendorStrike(models.Model):
    """نظام الإنذارات والعقوبات"""
    
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE, related_name='strikes')
    
    SEVERITY_CHOICES = [
        ('minor', 'خفيف'),
        ('moderate', 'متوسط'),
        ('major', 'شديد'),
        ('critical', 'حرج'),
    ]
    severity = models.CharField(max_length=20, choices=SEVERITY_CHOICES)
    
    REASON_CHOICES = [
        ('late_delivery', 'تأخير في التسليم'),
        ('high_cancellation', 'معدل إلغاء مرتفع'),
        ('poor_quality', 'جودة منتجات ضعيفة'),
        ('fake_products', 'منتجات مزيفة'),
        ('poor_service', 'خدمة عملاء سيئة'),
        ('policy_violation', 'مخالفة السياسات'),
        ('fraud', 'احتيال'),
    ]
    reason = models.CharField(max_length=50, choices=REASON_CHOICES)
    
    description = models.TextField()
    
    # العقوبة
    PENALTY_CHOICES = [
        ('warning', 'تحذير فقط'),
        ('listing_suspension', 'تعليق منتجات لمدة أسبوع'),
        ('account_restriction', 'تقييد الحساب لمدة شهر'),
        ('account_suspension', 'تعليق الحساب'),
        ('permanent_ban', 'حظر دائم'),
    ]
    penalty = models.CharField(max_length=30, choices=PENALTY_CHOICES, default='warning')
    
    # الحالة
    is_resolved = models.BooleanField(default=False)
    resolved_at = models.DateTimeField(null=True, blank=True)
    resolution_notes = models.TextField(blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    created_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    
    class Meta:
        ordering = ['-created_at']

class VendorMetrics(models.Model):
    """مقاييس أداء البائع (تحسب يومياً/أسبوعياً)"""
    
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE, related_name='metrics')
    
    # الفترة الزمنية
    period_start = models.DateField()
    period_end = models.DateField()
    
    # مقاييس الطلبات
    total_orders = models.IntegerField(default=0)
    completed_orders = models.IntegerField(default=0)
    cancelled_orders = models.IntegerField(default=0)
    cancellation_rate = models.FloatField(default=0)  # نسبة الإلغاء
    
    # مقاييس التسليم
    on_time_deliveries = models.IntegerField(default=0)
    late_deliveries = models.IntegerField(default=0)
    average_delivery_time = models.FloatField(default=0)  # بالأيام
    on_time_rate = models.FloatField(default=0)  # نسبة التسليم في الوقت
    
    # مقاييس الإرجاع
    total_returns = models.IntegerField(default=0)
    return_rate = models.FloatField(default=0)  # نسبة الإرجاع
    
    # مقاييس رضا العملاء
    total_reviews = models.IntegerField(default=0)
    average_rating = models.FloatField(default=0)
    positive_reviews = models.IntegerField(default=0)  # 4-5 نجوم
    negative_reviews = models.IntegerField(default=0)  # 1-2 نجوم
    
    # مقاييس الاستجابة
    average_response_time = models.FloatField(default=0)  # بالساعات
    response_rate = models.FloatField(default=0)  # نسبة الرد
    
    # المبيعات
    total_revenue = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        unique_together = ['vendor', 'period_start', 'period_end']
        ordering = ['-period_end']
```

### 3.2 خوارزمية حساب Performance Score

```python
# services/performance_calculator.py

from datetime import timedelta
from django.utils import timezone
from django.db.models import Avg, Count, Q

class PerformanceScoreCalculator:
    """حساب نقاط أداء البائع"""
    
    # الأوزان
    WEIGHTS = {
        'delivery': 0.30,        # 30% - التسليم في الوقت
        'cancellation': 0.20,    # 20% - نسبة الإلغاء
        'customer_service': 0.20, # 20% - خدمة العملاء
        'product_quality': 0.20,  # 20% - جودة المنتجات (التقييمات)
        'return_rate': 0.10,      # 10% - نسبة الإرجاع
    }
    
    @classmethod
    def calculate_vendor_score(cls, vendor_id: int) -> Dict:
        """حساب النقاط الشاملة للبائع"""
        
        # جلب بيانات آخر 30 يوم
        thirty_days_ago = timezone.now() - timedelta(days=30)
        
        metrics = VendorMetrics.objects.filter(
            vendor_id=vendor_id,
            period_start__gte=thirty_days_ago
        ).aggregate(
            total_orders=models.Sum('total_orders'),
            completed_orders=models.Sum('completed_orders'),
            cancelled_orders=models.Sum('cancelled_orders'),
            on_time_deliveries=models.Sum('on_time_deliveries'),
            late_deliveries=models.Sum('late_deliveries'),
            total_returns=models.Sum('total_returns'),
            total_reviews=models.Sum('total_reviews'),
            avg_rating=Avg('average_rating'),
            avg_response_time=Avg('average_response_time'),
            response_rate=Avg('response_rate'),
        )
        
        # 1. نقاط التسليم (0-100)
        if metrics['total_orders'] > 0:
            on_time_rate = (metrics['on_time_deliveries'] or 0) / metrics['total_orders']
            delivery_score = on_time_rate * 100
        else:
            delivery_score = 100  # بائع جديد = نقاط كاملة
        
        # 2. نقاط الإلغاء (0-100)
        if metrics['total_orders'] > 0:
            cancellation_rate = (metrics['cancelled_orders'] or 0) / metrics['total_orders']
            cancellation_score = max(0, 100 - (cancellation_rate * 200))  # كل 1% إلغاء = -2 نقطة
        else:
            cancellation_score = 100
        
        # 3. نقاط خدمة العملاء (0-100)
        response_rate = metrics['response_rate'] or 100
        response_time = metrics['avg_response_time'] or 0
        
        # الاستجابة السريعة = نقاط أعلى
        if response_time <= 1:  # أقل من ساعة
            response_time_score = 100
        elif response_time <= 4:  # أقل من 4 ساعات
            response_time_score = 80
        elif response_time <= 24:  # أقل من يوم
            response_time_score = 60
        else:
            response_time_score = 30
        
        customer_service_score = (response_rate * 0.5) + (response_time_score * 0.5)
        
        # 4. نقاط جودة المنتجات (0-100)
        avg_rating = metrics['avg_rating'] or 5.0
        product_quality_score = (avg_rating / 5.0) * 100
        
        # 5. نقاط نسبة الإرجاع (0-100)
        if metrics['completed_orders'] > 0:
            return_rate = (metrics['total_returns'] or 0) / metrics['completed_orders']
            return_rate_score = max(0, 100 - (return_rate * 300))  # كل 1% إرجاع = -3 نقاط
        else:
            return_rate_score = 100
        
        # حساب النقاط الإجمالية
        overall_score = (
            delivery_score * cls.WEIGHTS['delivery'] +
            cancellation_score * cls.WEIGHTS['cancellation'] +
            customer_service_score * cls.WEIGHTS['customer_service'] +
            product_quality_score * cls.WEIGHTS['product_quality'] +
            return_rate_score * cls.WEIGHTS['return_rate']
        )
        
        # تحديد الحالة
        if overall_score >= 90:
            status = 'excellent'
        elif overall_score >= 70:
            status = 'good'
        elif overall_score >= 50:
            status = 'average'
        elif overall_score >= 30:
            status = 'poor'
        else:
            status = 'critical'
        
        # حفظ النتائج
        performance, _ = VendorPerformanceScore.objects.update_or_create(
            vendor_id=vendor_id,
            defaults={
                'overall_score': round(overall_score, 2),
                'delivery_score': round(delivery_score, 2),
                'cancellation_score': round(cancellation_score, 2),
                'customer_service_score': round(customer_service_score, 2),
                'product_quality_score': round(product_quality_score, 2),
                'return_rate_score': round(return_rate_score, 2),
                'status': status,
            }
        )
        
        # التحقق من الحاجة لإنذار
        cls.check_for_strikes(vendor_id, overall_score, metrics)
        
        return {
            'overall_score': overall_score,
            'status': status,
            'breakdown': {
                'delivery': delivery_score,
                'cancellation': cancellation_score,
                'customer_service': customer_service_score,
                'product_quality': product_quality_score,
                'return_rate': return_rate_score,
            }
        }
    
    @classmethod
    def check_for_strikes(cls, vendor_id: int, overall_score: float, metrics: Dict):
        """التحقق من الحاجة لإصدار إنذار"""
        
        vendor = VendorProfile.objects.get(id=vendor_id)
        
        # إنذار إذا النقاط أقل من 30
        if overall_score < 30:
            VendorStrike.objects.create(
                vendor=vendor,
                severity='critical',
                reason='poor_performance',
                description=f'النقاط الإجمالية: {overall_score:.2f}/100 - أداء ضعيف جداً',
                penalty='account_restriction'
            )
        
        # إنذار إذا نسبة الإلغاء أكثر من 15%
        if metrics['total_orders'] > 10:
            cancellation_rate = (metrics['cancelled_orders'] or 0) / metrics['total_orders']
            if cancellation_rate > 0.15:
                VendorStrike.objects.create(
                    vendor=vendor,
                    severity='major',
                    reason='high_cancellation',
                    description=f'نسبة الإلغاء: {cancellation_rate*100:.1f}%',
                    penalty='listing_suspension'
                )
        
        # إنذار إذا نسبة الإرجاع أكثر من 20%
        if metrics['completed_orders'] > 10:
            return_rate = (metrics['total_returns'] or 0) / metrics['completed_orders']
            if return_rate > 0.20:
                VendorStrike.objects.create(
                    vendor=vendor,
                    severity='moderate',
                    reason='poor_quality',
                    description=f'نسبة الإرجاع: {return_rate*100:.1f}%',
                    penalty='warning'
                )
```

### 3.3 Celery Tasks للمراقبة التلقائية

```python
# tasks.py

@shared_task
def calculate_all_vendor_scores():
    """حساب نقاط جميع البائعين النشطين"""
    
    vendors = VendorProfile.objects.filter(is_active=True, is_approved=True)
    
    for vendor in vendors:
        try:
            PerformanceScoreCalculator.calculate_vendor_score(vendor.id)
        except Exception as e:
            logger.error(f"Error calculating score for vendor {vendor.id}: {e}")
    
    return f"Calculated scores for {vendors.count()} vendors"

@shared_task
def collect_vendor_metrics_daily():
    """جمع المقاييس اليومية لجميع البائعين"""
    
    today = timezone.now().date()
    yesterday = today - timedelta(days=1)
    
    vendors = VendorProfile.objects.filter(is_active=True)
    
    for vendor in vendors:
        # جمع بيانات الأمس
        orders = Order.objects.filter(
            vendor=vendor,
            created_at__date=yesterday
        )
        
        # ... حساب المقاييس وحفظها في VendorMetrics
```

### 3.4 لوحة تحكم الأداء للبائع

```typescript
// components/vendor/PerformancePanel.tsx

export default function PerformancePanel({ vendorId }: { vendorId: string }) {
  const { data: performance } = useQuery(['performance', vendorId], 
    () => fetchPerformance(vendorId)
  );
  
  if (!performance) return <Skeleton />;
  
  return (
    <Card>
      <CardHeader>
        <h2 className="text-2xl font-bold">نقاط الأداء</h2>
      </CardHeader>
      
      <CardContent className="space-y-6">
        {/* النقاط الإجمالية */}
        <div className="text-center">
          <div className="inline-flex items-center justify-center w-32 h-32 rounded-full border-8"
               style={{ borderColor: getScoreColor(performance.overall_score) }}>
            <div>
              <p className="text-4xl font-bold">{performance.overall_score}</p>
              <p className="text-sm text-gray-500">من 100</p>
            </div>
          </div>
          <p className="mt-4 text-lg font-medium">
            {getStatusLabel(performance.status)}
          </p>
        </div>
        
        {/* التفاصيل */}
        <div className="space-y-4">
          <PerformanceBar 
            label="التسليم في الوقت"
            score={performance.delivery_score}
            icon={<TruckIcon />}
          />
          
          <PerformanceBar 
            label="نسبة الإلغاء"
            score={performance.cancellation_score}
            icon={<XIcon />}
          />
          
          <PerformanceBar 
            label="خدمة العملاء"
            score={performance.customer_service_score}
            icon={<MessageIcon />}
          />
          
          <PerformanceBar 
            label="جودة المنتجات"
            score={performance.product_quality_score}
            icon={<StarIcon />}
          />
          
          <PerformanceBar 
            label="نسبة الإرجاع"
            score={performance.return_rate_score}
            icon={<ReturnIcon />}
          />
        </div>
        
        {/* الإنذارات */}
        {performance.strikes && performance.strikes.length > 0 && (
          <Alert variant="destructive">
            <AlertTriangle className="h-4 w-4" />
            <AlertTitle>لديك {performance.strikes.length} إنذار</AlertTitle>
            <AlertDescription>
              <ul className="mt-2 space-y-1">
                {performance.strikes.map((strike) => (
                  <li key={strike.id}>• {strike.description}</li>
                ))}
              </ul>
            </AlertDescription>
          </Alert>
        )}
        
        {/* نصائح التحسين */}
        <Card className="bg-blue-50 border-blue-200">
          <CardHeader>
            <h3 className="font-semibold">💡 نصائح لتحسين الأداء</h3>
          </CardHeader>
          <CardContent>
            <ul className="space-y-2 text-sm">
              {performance.overall_score < 70 && performance.delivery_score < 70 && (
                <li>• احرص على الشحن في الوقت المحدد - التأخير يؤثر على النقاط</li>
              )}
              {performance.cancellation_score < 70 && (
                <li>• قلل نسبة إلغاء الطلبات - تأكد من توفر المنتجات قبل القبول</li>
              )}
              {performance.product_quality_score < 70 && (
                <li>• حسّن جودة المنتجات - التقييمات السيئة تضر بسمعتك</li>
              )}
            </ul>
          </CardContent>
        </Card>
      </CardContent>
    </Card>
  );
}

function PerformanceBar({ label, score, icon }) {
  return (
    <div>
      <div className="flex items-center justify-between mb-2">
        <div className="flex items-center gap-2">
          {icon}
          <span className="text-sm font-medium">{label}</span>
        </div>
        <span className="text-sm font-bold">{score}/100</span>
      </div>
      <Progress value={score} className="h-2" />
    </div>
  );
}
```

---

## 4. التكامل التقني

### 4.1 الجداول الزمنية للمهام التلقائية

```python
# celery.py - Celery Beat Schedule

from celery.schedules import crontab

app.conf.beat_schedule = {
    # إعادة حساب Buy Box كل ساعة
    'recalculate-buy-box-hourly': {
        'task': 'marketplace.tasks.recalculate_buy_box_winners',
        'schedule': crontab(minute=0),  # كل ساعة
    },
    
    # حساب نقاط الأداء يومياً
    'calculate-performance-daily': {
        'task': 'vendors.tasks.calculate_all_vendor_scores',
        'schedule': crontab(hour=2, minute=0),  # 2 صباحاً يومياً
    },
    
    # جمع المقاييس اليومية
    'collect-metrics-daily': {
        'task': 'vendors.tasks.collect_vendor_metrics_daily',
        'schedule': crontab(hour=1, minute=0),  # 1 صباحاً يومياً
    },
    
    # مزامنة المنتجات مع الأسواق الخارجية كل 6 ساعات
    'sync-external-marketplaces': {
        'task': 'marketplace_integration.tasks.sync_all_vendor_marketplaces',
        'schedule': crontab(minute=0, hour='*/6'),  # كل 6 ساعات
    },
}
```

### 4.2 APIs الشاملة

```python
# urls.py

from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'vendor/marketplaces', VendorMarketplaceViewSet, basename='vendor-marketplaces')
router.register(r'vendor/performance', VendorPerformanceViewSet, basename='vendor-performance')
router.register(r'products/competition', ProductCompetitionViewSet, basename='product-competition')

urlpatterns = [
    path('api/', include(router.urls)),
]
```

---

## الخلاصة

### ✅ ما تم تغطيته:

1. **اختيار الأسواق للبائع:**
   - ✅ تفعيل/تعطيل أسواق محددة
   - ✅ استراتيجيات التسعير المختلفة
   - ✅ مزامنة تلقائية للمنتجات والمخزون
   - ✅ تتبع الأداء لكل سوق

2. **آلية المنافسة (Buy Box):**
   - ✅ نظام نقاط ذكي يفاضل بين البائعين
   - ✅ معايير متعددة: السعر، التقييم، الشحن، المخزون
   - ✅ إعادة حساب تلقائية كل ساعة
   - ✅ عرض جميع الخيارات للعميل

3. **مراقبة جودة البائعين:**
   - ✅ نظام نقاط شامل (0-100)
   - ✅ نظام إنذارات وعقوبات
   - ✅ مقاييس يومية تفصيلية
   - ✅ تحليل الأداء وتحسينه

### 🚀 الخطوات التالية:
1. مراجعة المواصفات
2. الموافقة على الآليات
3. البدء بالتطوير حسب الأولويات

---

**آخر تحديث:** 27 نوفمبر 2025
**الإصدار:** 1.0
**الحالة:** ✅ جاهز للمراجعة
