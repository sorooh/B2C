# 🌍 نظام الأسواق الجغرافية - Geographic Markets System
## البائع يختار الدول التي يريد البيع فيها

---

## 📋 الفرق الأساسي

### ❌ ما هو **ليس** هنا:
- **الأسواق الخارجية** (Amazon, Bol.com, eBay) → **الإدارة فقط تتحكم بها**
- نحن كمنصة GlobalMarket نبيع على تلك الأسواق
- البائع لا يعرف ولا يهتم

### ✅ ما هو **هنا**:
- **الأسواق الجغرافية** (هولندا, ألمانيا, فرنسا, مصر, الإمارات, السعودية, إلخ)
- البائع يختار الدول التي يريد الشحن إليها
- البائع يحدد سياسات الشحن والأسعار لكل دولة

---

## 🏗️ الهيكل التقني

### 1. Models - قواعد البيانات

```python
# models.py

from django.db import models
from django.contrib.postgres.fields import ArrayField

class Country(models.Model):
    """الدول المتاحة في المنصة"""
    
    code = models.CharField(max_length=2, unique=True)  # ISO 3166-1 alpha-2 (NL, DE, EG, etc.)
    name_en = models.CharField(max_length=100)
    name_ar = models.CharField(max_length=100)
    
    # معلومات إضافية
    currency = models.CharField(max_length=3)  # EUR, EGP, AED, etc.
    currency_symbol = models.CharField(max_length=5)
    language = models.CharField(max_length=5)  # nl, de, ar, etc.
    
    # المنطقة
    REGION_CHOICES = [
        ('europe_west', 'أوروبا الغربية'),
        ('europe_east', 'أوروبا الشرقية'),
        ('middle_east', 'الشرق الأوسط'),
        ('north_africa', 'شمال أفريقيا'),
        ('gcc', 'دول الخليج'),
        ('north_america', 'أمريكا الشمالية'),
        ('asia', 'آسيا'),
    ]
    region = models.CharField(max_length=20, choices=REGION_CHOICES)
    
    # الحالة
    is_active = models.BooleanField(default=True)
    is_eu_member = models.BooleanField(default=False)
    
    # معدل الضريبة (VAT/GST)
    tax_rate = models.DecimalField(max_digits=5, decimal_places=2, default=0)
    
    # معلومات الشحن الافتراضية
    default_shipping_cost = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    average_delivery_days = models.IntegerField(default=5)
    
    class Meta:
        verbose_name = 'دولة'
        verbose_name_plural = 'الدول'
        ordering = ['name_en']

class VendorGeographicMarket(models.Model):
    """الأسواق الجغرافية التي يبيع فيها البائع"""
    
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE, related_name='geographic_markets')
    country = models.ForeignKey(Country, on_delete=models.CASCADE)
    
    # تفعيل السوق
    is_enabled = models.BooleanField(default=True)
    
    # سياسة الشحن لهذا السوق
    SHIPPING_POLICY_CHOICES = [
        ('free', 'شحن مجاني'),
        ('flat_rate', 'سعر ثابت'),
        ('calculated', 'محسوب حسب الوزن'),
        ('not_available', 'لا يتوفر شحن'),
    ]
    shipping_policy = models.CharField(max_length=20, choices=SHIPPING_POLICY_CHOICES, default='flat_rate')
    
    # تكلفة الشحن (إذا كانت flat_rate)
    shipping_cost = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    
    # شحن مجاني عند شراء بمبلغ معين
    free_shipping_threshold = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    # وقت التوصيل المتوقع
    estimated_delivery_days_min = models.IntegerField(default=3)
    estimated_delivery_days_max = models.IntegerField(default=7)
    
    # معلومات الجمارك (للأسواق الدولية)
    includes_customs_fees = models.BooleanField(default=False)
    customs_info = models.TextField(blank=True)
    
    # أولوية السوق (1 = أعلى أولوية)
    priority = models.IntegerField(default=5)
    
    # الإحصائيات
    total_orders = models.IntegerField(default=0)
    total_revenue = models.DecimalField(max_digits=12, decimal_places=2, default=0)
    average_rating = models.FloatField(default=0)
    
    # ملاحظات خاصة بالبائع
    notes = models.TextField(blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        unique_together = ['vendor', 'country']
        verbose_name = 'سوق جغرافي'
        verbose_name_plural = 'الأسواق الجغرافية'
        ordering = ['priority', 'country__name_en']

class VendorShippingZone(models.Model):
    """مناطق شحن مخصصة للبائع (اختياري - للتحكم المتقدم)"""
    
    vendor = models.ForeignKey(VendorProfile, on_delete=models.CASCADE, related_name='shipping_zones')
    
    name = models.CharField(max_length=100)  # مثل: "دول الخليج", "أوروبا", "شمال أفريقيا"
    
    # الدول المشمولة في هذه المنطقة
    countries = models.ManyToManyField(Country)
    
    # سياسة الشحن الموحدة لهذه المنطقة
    shipping_cost = models.DecimalField(max_digits=10, decimal_places=2)
    free_shipping_threshold = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    estimated_delivery_days_min = models.IntegerField(default=3)
    estimated_delivery_days_max = models.IntegerField(default=10)
    
    is_active = models.BooleanField(default=True)
    
    class Meta:
        verbose_name = 'منطقة شحن'
        verbose_name_plural = 'مناطق الشحن'

class ProductGeographicAvailability(models.Model):
    """توفر المنتج في أسواق جغرافية معينة"""
    
    product = models.ForeignKey(Product, on_delete=models.CASCADE, related_name='geographic_availability')
    country = models.ForeignKey(Country, on_delete=models.CASCADE)
    
    # التوفر
    is_available = models.BooleanField(default=True)
    
    # سعر مخصص لهذا السوق (اختياري)
    custom_price = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    
    # مخزون مخصص لهذا السوق (اختياري)
    custom_stock = models.IntegerField(null=True, blank=True)
    
    # قيود
    requires_special_permit = models.BooleanField(default=False)
    permit_info = models.TextField(blank=True)
    
    # ملاحظات
    notes = models.TextField(blank=True)
    
    class Meta:
        unique_together = ['product', 'country']
        verbose_name = 'توفر جغرافي'
        verbose_name_plural = 'التوفر الجغرافي'
```

---

## 🎨 واجهة المستخدم - Vendor Dashboard

### 1. صفحة إعدادات الأسواق الجغرافية

```typescript
// app/vendor/markets/page.tsx

import { useState, useEffect } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';

interface GeographicMarket {
  id: number;
  country: Country;
  is_enabled: boolean;
  shipping_policy: string;
  shipping_cost: number;
  free_shipping_threshold: number | null;
  estimated_delivery_days_min: number;
  estimated_delivery_days_max: number;
  total_orders: number;
  total_revenue: number;
}

export default function GeographicMarketsPage() {
  const { data: markets, isLoading } = useQuery<GeographicMarket[]>(
    ['vendor-markets'],
    fetchVendorMarkets
  );
  
  const { data: availableCountries } = useQuery<Country[]>(
    ['available-countries'],
    fetchAvailableCountries
  );
  
  const [selectedRegion, setSelectedRegion] = useState<string>('all');
  
  if (isLoading) return <LoadingSkeleton />;
  
  // تصنيف الدول حسب المنطقة
  const countriesByRegion = availableCountries?.reduce((acc, country) => {
    if (!acc[country.region]) acc[country.region] = [];
    acc[country.region].push(country);
    return acc;
  }, {} as Record<string, Country[]>);
  
  return (
    <div className="p-6 space-y-6">
      {/* Header */}
      <div>
        <h1 className="text-3xl font-bold">الأسواق الجغرافية</h1>
        <p className="text-gray-600 mt-2">
          اختر الدول التي تريد الشحن إليها وحدد سياسات الشحن لكل سوق
        </p>
      </div>
      
      {/* Quick Stats */}
      <div className="grid grid-cols-4 gap-4">
        <StatCard
          title="الأسواق المفعلة"
          value={markets?.filter(m => m.is_enabled).length || 0}
          icon={<GlobeIcon />}
          color="blue"
        />
        <StatCard
          title="إجمالي الطلبات"
          value={markets?.reduce((sum, m) => sum + m.total_orders, 0) || 0}
          icon={<ShoppingBagIcon />}
          color="green"
        />
        <StatCard
          title="إجمالي الإيرادات"
          value={`€${markets?.reduce((sum, m) => sum + m.total_revenue, 0).toFixed(2) || 0}`}
          icon={<CurrencyIcon />}
          color="purple"
        />
        <StatCard
          title="معدل الشحن"
          value={`€${(markets?.reduce((sum, m) => sum + m.shipping_cost, 0) / markets?.length || 0).toFixed(2)}`}
          icon={<TruckIcon />}
          color="orange"
        />
      </div>
      
      {/* Region Filter */}
      <Card>
        <CardContent className="p-4">
          <div className="flex gap-2 flex-wrap">
            <Badge
              variant={selectedRegion === 'all' ? 'default' : 'outline'}
              className="cursor-pointer"
              onClick={() => setSelectedRegion('all')}
            >
              جميع المناطق ({availableCountries?.length || 0})
            </Badge>
            {Object.entries(countriesByRegion || {}).map(([region, countries]) => (
              <Badge
                key={region}
                variant={selectedRegion === region ? 'default' : 'outline'}
                className="cursor-pointer"
                onClick={() => setSelectedRegion(region)}
              >
                {getRegionLabel(region)} ({countries.length})
              </Badge>
            ))}
          </div>
        </CardContent>
      </Card>
      
      {/* Markets Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {(selectedRegion === 'all' 
          ? availableCountries 
          : countriesByRegion?.[selectedRegion] || []
        ).map((country) => {
          const market = markets?.find(m => m.country.code === country.code);
          return (
            <CountryMarketCard
              key={country.code}
              country={country}
              market={market}
            />
          );
        })}
      </div>
    </div>
  );
}

function CountryMarketCard({ country, market }: { country: Country; market?: GeographicMarket }) {
  const [isExpanded, setIsExpanded] = useState(false);
  const toggleMarket = useMutation(toggleCountryMarket);
  const updateMarket = useMutation(updateCountryMarketSettings);
  
  const isEnabled = market?.is_enabled || false;
  
  return (
    <Card className={isEnabled ? 'border-green-500 border-2' : 'border-gray-200'}>
      <CardHeader>
        <div className="flex items-center justify-between">
          <div className="flex items-center gap-3">
            {/* علم الدولة */}
            <div className="text-4xl">{getFlagEmoji(country.code)}</div>
            
            <div>
              <h3 className="font-semibold text-lg">{country.name_ar}</h3>
              <p className="text-sm text-gray-500">{country.name_en}</p>
              <Badge variant="secondary" className="mt-1 text-xs">
                {getRegionLabel(country.region)}
              </Badge>
            </div>
          </div>
          
          {/* مفتاح التفعيل */}
          <Switch
            checked={isEnabled}
            onCheckedChange={() => toggleMarket.mutate(country.code)}
          />
        </div>
      </CardHeader>
      
      {isEnabled && (
        <CardContent className="space-y-4">
          {/* سياسة الشحن */}
          <div>
            <Label className="text-sm font-medium">سياسة الشحن</Label>
            <Select
              value={market?.shipping_policy || 'flat_rate'}
              onValueChange={(value) => updateMarket.mutate({
                country: country.code,
                shipping_policy: value
              })}
            >
              <SelectTrigger className="mt-1">
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="free">شحن مجاني 🎁</SelectItem>
                <SelectItem value="flat_rate">سعر ثابت 📦</SelectItem>
                <SelectItem value="calculated">محسوب حسب الوزن ⚖️</SelectItem>
                <SelectItem value="not_available">لا يتوفر شحن ❌</SelectItem>
              </SelectContent>
            </Select>
          </div>
          
          {/* تكلفة الشحن */}
          {market?.shipping_policy === 'flat_rate' && (
            <div>
              <Label className="text-sm font-medium">تكلفة الشحن</Label>
              <div className="flex gap-2 mt-1">
                <Input
                  type="number"
                  step="0.01"
                  value={market.shipping_cost}
                  onChange={(e) => updateMarket.mutate({
                    country: country.code,
                    shipping_cost: parseFloat(e.target.value)
                  })}
                  className="flex-1"
                />
                <span className="flex items-center px-3 bg-gray-100 rounded-md">
                  {country.currency_symbol}
                </span>
              </div>
            </div>
          )}
          
          {/* شحن مجاني عند */}
          <div>
            <Label className="text-sm font-medium">شحن مجاني عند الشراء بـ</Label>
            <div className="flex gap-2 mt-1">
              <Input
                type="number"
                step="0.01"
                value={market?.free_shipping_threshold || ''}
                onChange={(e) => updateMarket.mutate({
                  country: country.code,
                  free_shipping_threshold: parseFloat(e.target.value) || null
                })}
                placeholder="اختياري"
                className="flex-1"
              />
              <span className="flex items-center px-3 bg-gray-100 rounded-md">
                {country.currency_symbol}
              </span>
            </div>
          </div>
          
          {/* وقت التوصيل */}
          <div>
            <Label className="text-sm font-medium">وقت التوصيل المتوقع</Label>
            <div className="flex gap-2 items-center mt-1">
              <Input
                type="number"
                value={market?.estimated_delivery_days_min || 3}
                onChange={(e) => updateMarket.mutate({
                  country: country.code,
                  estimated_delivery_days_min: parseInt(e.target.value)
                })}
                className="w-20"
              />
              <span className="text-sm">إلى</span>
              <Input
                type="number"
                value={market?.estimated_delivery_days_max || 7}
                onChange={(e) => updateMarket.mutate({
                  country: country.code,
                  estimated_delivery_days_max: parseInt(e.target.value)
                })}
                className="w-20"
              />
              <span className="text-sm text-gray-500">أيام</span>
            </div>
          </div>
          
          {/* الإحصائيات */}
          {market && market.total_orders > 0 && (
            <div className="pt-4 border-t">
              <p className="text-sm font-medium mb-2">الأداء في هذا السوق:</p>
              <div className="grid grid-cols-2 gap-2 text-sm">
                <div>
                  <p className="text-gray-500">الطلبات</p>
                  <p className="font-bold">{market.total_orders}</p>
                </div>
                <div>
                  <p className="text-gray-500">الإيرادات</p>
                  <p className="font-bold">{country.currency_symbol}{market.total_revenue.toFixed(2)}</p>
                </div>
              </div>
            </div>
          )}
          
          {/* إعدادات متقدمة */}
          <Button
            variant="outline"
            size="sm"
            className="w-full"
            onClick={() => setIsExpanded(!isExpanded)}
          >
            إعدادات متقدمة
          </Button>
          
          {isExpanded && (
            <div className="space-y-3 pt-3 border-t">
              {/* ملاحظات خاصة */}
              <div>
                <Label className="text-sm font-medium">ملاحظات</Label>
                <Textarea
                  value={market?.notes || ''}
                  onChange={(e) => updateMarket.mutate({
                    country: country.code,
                    notes: e.target.value
                  })}
                  placeholder="ملاحظات خاصة بهذا السوق..."
                  className="mt-1"
                  rows={3}
                />
              </div>
              
              {/* معلومات الضريبة */}
              <div className="bg-blue-50 p-3 rounded-md">
                <p className="text-sm font-medium">معلومات الضريبة</p>
                <p className="text-xs text-gray-600 mt-1">
                  معدل الضريبة في {country.name_ar}: {country.tax_rate}%
                </p>
              </div>
            </div>
          )}
        </CardContent>
      )}
    </Card>
  );
}

// Helper functions
function getFlagEmoji(countryCode: string): string {
  const codePoints = countryCode
    .toUpperCase()
    .split('')
    .map(char => 127397 + char.charCodeAt(0));
  return String.fromCodePoint(...codePoints);
}

function getRegionLabel(region: string): string {
  const labels: Record<string, string> = {
    'europe_west': 'أوروبا الغربية',
    'europe_east': 'أوروبا الشرقية',
    'middle_east': 'الشرق الأوسط',
    'north_africa': 'شمال أفريقيا',
    'gcc': 'دول الخليج',
    'north_america': 'أمريكا الشمالية',
    'asia': 'آسيا',
  };
  return labels[region] || region;
}
```

---

## 🛒 تجربة العميل - اختيار الدولة

### 1. سياسة الشحن تظهر حسب دولة العميل

```typescript
// components/product/ShippingInfo.tsx

export default function ShippingInfo({ product, selectedCountry }: Props) {
  const { data: shippingInfo } = useQuery(
    ['shipping-info', product.id, selectedCountry],
    () => fetchShippingInfo(product.id, selectedCountry)
  );
  
  if (!shippingInfo || !shippingInfo.available) {
    return (
      <Alert variant="destructive">
        <AlertTriangle className="h-4 w-4" />
        <AlertTitle>غير متوفر للشحن</AlertTitle>
        <AlertDescription>
          هذا المنتج غير متوفر للشحن إلى {selectedCountry}
        </AlertDescription>
      </Alert>
    );
  }
  
  return (
    <Card className="bg-green-50 border-green-200">
      <CardContent className="p-4">
        <div className="flex items-start gap-3">
          <TruckIcon className="h-5 w-5 text-green-600 mt-1" />
          
          <div className="flex-1">
            <h3 className="font-semibold text-green-900">متوفر للشحن إلى {selectedCountry}</h3>
            
            <div className="mt-2 space-y-1 text-sm">
              {/* تكلفة الشحن */}
              {shippingInfo.shipping_policy === 'free' ? (
                <p className="text-green-700 font-medium">
                  🎁 شحن مجاني
                </p>
              ) : (
                <p>
                  تكلفة الشحن: 
                  <span className="font-bold ml-1">
                    {shippingInfo.shipping_cost} {shippingInfo.currency}
                  </span>
                </p>
              )}
              
              {/* شحن مجاني عند */}
              {shippingInfo.free_shipping_threshold && (
                <p className="text-green-700">
                  ✨ شحن مجاني عند الشراء بـ {shippingInfo.free_shipping_threshold} {shippingInfo.currency}
                </p>
              )}
              
              {/* وقت التوصيل */}
              <p className="flex items-center gap-1">
                <ClockIcon className="h-4 w-4" />
                التوصيل خلال {shippingInfo.delivery_min}-{shippingInfo.delivery_max} أيام
              </p>
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

### 2. اختيار الدولة في Header

```typescript
// components/layout/CountrySelector.tsx

export default function CountrySelector() {
  const [selectedCountry, setSelectedCountry] = useAtom(selectedCountryAtom);
  const { data: countries } = useQuery(['countries'], fetchAvailableCountries);
  
  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="sm" className="gap-2">
          <span className="text-xl">{getFlagEmoji(selectedCountry)}</span>
          <span>{getCountryName(selectedCountry)}</span>
          <ChevronDown className="h-4 w-4" />
        </Button>
      </DropdownMenuTrigger>
      
      <DropdownMenuContent align="end" className="w-64">
        <DropdownMenuLabel>اختر دولتك</DropdownMenuLabel>
        <DropdownMenuSeparator />
        
        {/* تجميع حسب المنطقة */}
        {Object.entries(groupCountriesByRegion(countries)).map(([region, regionCountries]) => (
          <div key={region}>
            <DropdownMenuLabel className="text-xs text-gray-500">
              {getRegionLabel(region)}
            </DropdownMenuLabel>
            {regionCountries.map((country) => (
              <DropdownMenuItem
                key={country.code}
                onClick={() => setSelectedCountry(country.code)}
                className="flex items-center gap-2"
              >
                <span className="text-xl">{getFlagEmoji(country.code)}</span>
                <span>{country.name_ar}</span>
                {selectedCountry === country.code && (
                  <Check className="h-4 w-4 ml-auto" />
                )}
              </DropdownMenuItem>
            ))}
          </div>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

---

## 📊 APIs

```python
# views.py

from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response

class VendorGeographicMarketViewSet(viewsets.ModelViewSet):
    """إدارة الأسواق الجغرافية للبائع"""
    
    def get_queryset(self):
        return VendorGeographicMarket.objects.filter(
            vendor=self.request.user.vendor_profile
        ).select_related('country')
    
    @action(detail=False, methods=['get'])
    def available_countries(self, request):
        """قائمة الدول المتاحة"""
        countries = Country.objects.filter(is_active=True)
        
        # جلب الأسواق المفعلة للبائع
        vendor_markets = VendorGeographicMarket.objects.filter(
            vendor=request.user.vendor_profile
        ).values_list('country_id', flat=True)
        
        data = []
        for country in countries:
            data.append({
                'code': country.code,
                'name_en': country.name_en,
                'name_ar': country.name_ar,
                'region': country.region,
                'currency': country.currency,
                'currency_symbol': country.currency_symbol,
                'tax_rate': float(country.tax_rate),
                'is_enabled': country.id in vendor_markets,
            })
        
        return Response(data)
    
    @action(detail=False, methods=['post'])
    def toggle_country(self, request):
        """تفعيل/تعطيل دولة معينة"""
        country_code = request.data.get('country_code')
        country = get_object_or_404(Country, code=country_code)
        
        market, created = VendorGeographicMarket.objects.get_or_create(
            vendor=request.user.vendor_profile,
            country=country,
            defaults={
                'shipping_cost': country.default_shipping_cost,
                'estimated_delivery_days_min': 3,
                'estimated_delivery_days_max': country.average_delivery_days,
            }
        )
        
        if not created:
            market.is_enabled = not market.is_enabled
            market.save()
        
        return Response({
            'status': 'success',
            'is_enabled': market.is_enabled,
        })
    
    @action(detail=False, methods=['put'])
    def update_market_settings(self, request):
        """تحديث إعدادات سوق معين"""
        country_code = request.data.get('country_code')
        country = get_object_or_404(Country, code=country_code)
        
        market = get_object_or_404(
            VendorGeographicMarket,
            vendor=request.user.vendor_profile,
            country=country
        )
        
        # تحديث الإعدادات
        serializer = GeographicMarketSerializer(market, data=request.data, partial=True)
        
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class ProductShippingAPIView(APIView):
    """API للعميل - الحصول على معلومات الشحن"""
    
    def get(self, request, product_id):
        country_code = request.query_params.get('country', 'NL')
        
        product = get_object_or_404(Product, id=product_id)
        country = get_object_or_404(Country, code=country_code)
        
        # التحقق من توفر المنتج في هذا السوق
        market = VendorGeographicMarket.objects.filter(
            vendor=product.vendor,
            country=country,
            is_enabled=True
        ).first()
        
        if not market:
            return Response({
                'available': False,
                'message': f'المنتج غير متوفر للشحن إلى {country.name_ar}'
            })
        
        return Response({
            'available': True,
            'country': {
                'code': country.code,
                'name': country.name_ar,
                'currency': country.currency,
                'currency_symbol': country.currency_symbol,
            },
            'shipping_policy': market.shipping_policy,
            'shipping_cost': float(market.shipping_cost),
            'free_shipping_threshold': float(market.free_shipping_threshold) if market.free_shipping_threshold else None,
            'delivery_min': market.estimated_delivery_days_min,
            'delivery_max': market.estimated_delivery_days_max,
        })
```

---

## 🔄 سيناريوهات الاستخدام

### السيناريو 1: بائع هولندي يريد البيع في أوروبا فقط

```yaml
الإعدادات:
  - هولندا: شحن مجاني
  - بلجيكا: €3.99
  - ألمانيا: €5.99
  - فرنسا: €7.99
  - باقي أوروبا: غير متوفر
  - الشرق الأوسط: غير متوفر
```

### السيناريو 2: بائع مصري يريد البيع في الشرق الأوسط

```yaml
الإعدادات:
  - مصر: شحن مجاني
  - السعودية: 50 ريال
  - الإمارات: 40 درهم
  - الكويت: 5 دينار
  - أوروبا: غير متوفر
```

### السيناريو 3: بائع عالمي

```yaml
الإعدادات:
  - أوروبا: €5-10 حسب الدولة
  - الشرق الأوسط: €15-20
  - أمريكا: €25
  - آسيا: €20
```

---

## ✅ الخلاصة

### الفرق الواضح:

| الميزة | الأسواق الخارجية | الأسواق الجغرافية |
|--------|-------------------|-------------------|
| **ما هي؟** | Amazon, Bol.com, eBay | هولندا, مصر, الإمارات |
| **من يتحكم؟** | **الإدارة فقط** | **البائع** |
| **الغرض** | نحن نبيع كمنصة | البائع يشحن للعملاء |
| **يظهر للبائع؟** | ❌ لا | ✅ نعم |
| **الإعدادات** | API credentials، مزامنة | سياسات الشحن، الأسعار |

---

**آخر تحديث:** 28 نوفمبر 2025  
**الإصدار:** 1.0  
**الحالة:** ✅ جاهز للتطوير
