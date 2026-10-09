# Orchard Core'da "Shape" Kavramı

## Kısaca

**Shape**, Orchard Core'da ekranda render edilecek her şeyi temsil eden **dinamik bir view model** nesnesidir.
Controller veya driver doğrudan HTML üretmez; bunun yerine bir shape oluşturur, ona veri ekler ve
"bunu render et" der. Hangi şablonun (template) kullanılacağına ise **render anında** karar verilir.

> Shape = Veri (model) + Tip adı + Meta veriler (alternates, wrappers, zone, sıra...)
> Template (Razor `.cshtml` / Liquid `.liquid`) = Shape'in nasıl görüneceği

Bu ayrım sayesinde modül kodu hiç değiştirilmeden, sadece temada bir şablon ekleyerek görünüm değiştirilebilir.

## Neden var?

Klasik ASP.NET MVC'de `return View("Product", model)` dediğinizde view dosyası sabittir.
Orchard'da ise:

- İçerik **parçalardan** (Part) ve **alanlardan** (Field) oluşur; her biri kendi shape'ini üretir.
- Aynı içerik farklı yerlerde farklı görünmelidir (`Detail`, `Summary`, `SummaryAdmin`...).
- Temalar, modüllerin ürettiği çıktıyı **override** edebilmelidir.

Shape bu esnekliği sağlayan soyutlama katmanıdır.

## Temel bileşenler

| Kavram | Açıklama |
|---|---|
| **Shape Type** | Shape'in adı, ör. `Content`, `TitlePart`, `MenuItem`. Varsayılan şablon adını belirler. |
| **Metadata** | `shape.Metadata` → `Type`, `DisplayType`, `Alternates`, `Wrappers`, `Position`, `Differentiator`, `PlacementSource`. |
| **Properties** | Shape'e eklenen veriler (dinamik veya `ShapeViewModel<T>` ile tipli). |
| **Alternates** | Aynı shape için denenecek alternatif şablon adları (daha spesifik olan kazanır). |
| **Wrappers** | Shape'i saran dış şablonlar (ör. `Widget_Wrapper`). |
| **Zones** | Shape içindeki isimli bölgeler (`Header`, `Content`, `Footer`, `Meta`...). Zone da bir shape'tir. |
| **Placement** | Hangi shape'in hangi zone'a, hangi sırayla gideceğini belirleyen kurallar (`placement.json` veya driver içi `.Location()`). |

## Yaşam döngüsü

```
Driver / Controller          IShapeFactory             IDisplayManager / Placement
      │  shape oluştur  ───►  Shape nesnesi  ───►  zone'lara yerleştir (Placement)
      │                                               │
      ▼                                               ▼
                   Shape Table (şablon arama)  ◄── Alternates listesi
                              │
                              ▼
          Razor/Liquid şablonu bulunur → HTML üretilir (Wrappers ile sarılır)
```

1. **Oluşturma:** `IShapeFactory.CreateAsync("TypeName", model)` veya driver'da `Initialize<TViewModel>("ShapeName", m => {...})`.
2. **Olaylar:** `IShapeTableProvider` / `ShapeTableBuilder` ile `OnCreated`, `OnDisplaying`, `OnDisplayed` olaylarına bağlanılır; burada alternate/wrapper eklenebilir.
3. **Yerleşim:** Placement kurallarıyla zone ve sıra belirlenir.
4. **Şablon çözümleme:** Alternates en spesifikten en genele denenir; ilk bulunan şablon kullanılır.
5. **Render:** Şablon çalışır; varsa wrapper'lar dışarıdan sarar.

## Örnek: Driver ile shape üretmek

```csharp
public class ProductPartDisplayDriver : ContentPartDisplayDriver<ProductPart>
{
    public override IDisplayResult Display(ProductPart part, BuildPartDisplayContext context)
    {
        return Initialize<ProductPartViewModel>("ProductPart", model =>
        {
            model.Price = part.Price;
            model.Sku   = part.Sku;
        })
        .Location("Detail", "Content:5")   // Detail görünümünde Content zone'una, 5. sıraya
        .Location("Summary", "Meta:1");    // Summary görünümünde Meta zone'una
    }
}
```

Şablon: `Views/ProductPart.cshtml`

```cshtml
@model ProductPartViewModel
<div class="product">
    <span class="sku">@Model.Sku</span>
    <strong class="price">@Model.Price.ToString("C")</strong>
</div>
```

## Alternates (şablon adlandırma)

Bir `BlogPost` içerik öğesi `Summary` görünümünde render edilirken `Content` shape'i için
yaklaşık şu alternatifler denenir (en spesifik olan önce kazanır):

```
Content_Summary__42          →  Content-42.Summary.cshtml   (belirli bir içerik id'si için)
Content__42                  →  Content-42.cshtml
Content_Summary__BlogPost    →  Content-BlogPost.Summary.cshtml
Content__BlogPost            →  Content-BlogPost.cshtml
Content_Summary              →  Content.Summary.cshtml
Content                      →  Content.cshtml
```

Dosya adı dönüşümü:
- `__` (çift alt çizgi) → `-` (tire)
- `_` (tek alt çizgi) → `.` (nokta)
- Display type her zaman dosya adının sonuna taşınır: `Content_Summary__BlogPost` → `Content-BlogPost.Summary`

Kod ile alternate eklemek:

```csharp
builder.Describe("Content")
    .OnDisplaying(ctx =>
    {
        if (ctx.Shape is IShape shape && ctx.DisplayType == "Summary")
        {
            shape.Metadata.Alternates.Add("Content__MyCustomSummary");
        }
    });
```

## Placement (`placement.json`)

Kod değiştirmeden shape'lerin yerini/görünürlüğünü temadan ayarlamak için:

```json
{
  "TitlePart": [
    { "displayType": "Detail",  "place": "Header:1" },
    { "displayType": "Summary", "place": "Header:1", "alternates": ["TitlePart_Summary"] }
  ],
  "ProductPart": [
    { "contentType": ["Product"], "displayType": "Summary", "place": "-" }
  ]
}
```

`"place": "-"` → shape render edilmez (gizlenir).

## Razor / Liquid içinde shape kullanımı

```cshtml
@* Zone'u render et *@
@await RenderSectionAsync("Header", required: false)
@await DisplayAsync(Model.Content)

@* Yeni bir shape oluşturup render et *@
@{
    var badge = await New.Badge(Text: "Yeni", Color: "green");
}
@await DisplayAsync(badge)

@* Tag helper ile *@
<shape type="Badge" prop-text="Yeni" prop-color="green" />
```

Liquid:

```liquid
{{ Model.Content | shape_render }}
{% shape "Badge", text: "Yeni", color: "green" %}
```

## Frontend geliştirici açısından özet

- Bir şeyin görünümünü değiştirmek istiyorsanız **önce shape adını bulun** (Admin → *Templates* modülü
  veya `Shape Tracing`/debug ile), sonra temanızda uygun **alternate adıyla** şablon oluşturun.
- Yerini/sırasını değiştirmek için **placement**, görünümünü değiştirmek için **template**, ek sarmalayıcı
  için **wrapper** kullanın.
- Modül koduna dokunmadan tema katmanında override etmek Orchard'ın temel felsefesidir.

## Kaynaklar

- Orchard Core Docs – Display Management / Shapes: https://docs.orchardcore.net/en/latest/reference/modules/DisplayManagement/
- Placement: https://docs.orchardcore.net/en/latest/reference/modules/Placements/
- Templates (Admin'den shape override): https://docs.orchardcore.net/en/latest/reference/modules/Templates/
