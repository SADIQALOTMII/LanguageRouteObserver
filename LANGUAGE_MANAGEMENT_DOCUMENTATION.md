# توثيق نظام إدارة اللغة والتحديث التلقائي للبيانات
## Language Management & Auto-Refresh System Documentation

---

## 📋 جدول المحتويات

1. [نظرة عامة](#نظرة-عامة)
2. [المشكلة التي يحلها النظام](#المشكلة-التي-يحلها-النظام)
3. [المكونات الرئيسية](#المكونات-الرئيسية)
4. [LanguageAwareWidget - شرح تفصيلي](#languageawarewidget)
5. [GlobalAppBar - شرح تفصيلي](#globalappbar)
6. [LanguageRouteObserver - شرح تفصيلي](#languagerouteobserver)
7. [كيف يعمل النظام معاً](#كيف-يعمل-النظام-معاً)
8. [أمثلة عملية من الكود](#أمثلة-عملية-من-الكود)
9. [سيناريوهات الاستخدام](#سيناريوهات-الاستخدام)
10. [أفضل الممارسات](#أفضل-الممارسات)

---

## نظرة عامة

نظام إدارة اللغة والتحديث التلقائي للبيانات هو نظام متكامل يضمن تحديث واجهة المستخدم والبيانات تلقائياً عند تغيير اللغة في أي مكان في التطبيق. يتكون النظام من ثلاثة مكونات رئيسية:

1. **`LanguageAwareWidget`**: Widget wrapper يستمع لتغييرات اللغة ويستدعي callbacks لإعادة تحميل البيانات
2. **`GlobalAppBar`**: AppBar موحد يحتوي على مبدل اللغة ويدير التحديث الفوري عند تغيير اللغة في الشاشة الحالية
3. **`LanguageRouteObserver`**: RouteObserver يتتبع حالة الشاشات (نشطة/غير نشطة) لتحديد متى يجب تحديث البيانات

---

## المشكلة التي يحلها النظام

### المشكلة الأساسية:

عندما يغير المستخدم اللغة في شاشة معينة (مثل شاشة تفاصيل الوثيقة)، ثم يعود إلى الشاشة السابقة (مثل شاشة الفروع)، يجب أن تتحدث الشاشة السابقة تلقائياً لتعكس اللغة الجديدة.

### السيناريو:

```
1. المستخدم في شاشة "الفروع" (اللغة: عربي)
2. المستخدم ينتقل إلى شاشة "تفاصيل الوثيقة"
3. المستخدم يغير اللغة إلى الإنجليزية في شاشة "تفاصيل الوثيقة"
4. المستخدم يعود إلى شاشة "الفروع"
5. يجب أن تظهر شاشة "الفروع" باللغة الإنجليزية تلقائياً
```

### الحل:

النظام يضمن:
- ✅ تحديث فوري عند تغيير اللغة في الشاشة الحالية (عبر `GlobalAppBar`)
- ✅ تحديث تلقائي عند العودة إلى شاشة بعد تغيير اللغة في شاشة أخرى (عبر `LanguageAwareWidget`)
- ✅ عدم تكرار التحديث (تجنب استدعاء callback مرتين)

---

## المكونات الرئيسية

### 1. LanguageAwareWidget

**الموقع**: `lib/core/widgets/language_aware_widget.dart`

**الوظيفة**: Widget wrapper يستمع لتغييرات اللغة ويستدعي callback لإعادة تحميل البيانات عند العودة إلى الشاشة بعد تغيير اللغة في شاشة أخرى.

### 2. GlobalAppBar

**الموقع**: `lib/app_modules/global_app_bar.dart`

**الوظيفة**: AppBar موحد يحتوي على:
- شعار التطبيق أو زر الرجوع
- رسالة الترحيب أو عنوان الشاشة
- مبدل اللغة
- زر الإشعارات
- يدير التحديث الفوري عند تغيير اللغة في الشاشة الحالية

### 3. LanguageRouteObserver

**الموقع**: `lib/core/navigation/language_route_observer.dart`

**الوظيفة**: RouteObserver يتتبع حالة الشاشات (نشطة/غير نشطة) لتحديد متى يجب تحديث البيانات.

---

## LanguageAwareWidget

### 📝 الوصف

`LanguageAwareWidget` هو `StatefulWidget` يغلف أي widget ويستمع تلقائياً لتغييرات اللغة عبر `BlocListener<LanguageCubit>`. يستخدم `RouteAware` mixin لتحديد ما إذا كانت الشاشة نشطة حالياً أم لا.

### 🔧 المعاملات (Parameters)

```dart
class LanguageAwareWidget extends StatefulWidget {
  /// Widget الذي سيتم تغليفه
  final Widget child;
  
  /// Callback يتم استدعاؤه عند تغيير اللغة بينما الشاشة في الخلفية
  /// ثم العودة إلى هذه الشاشة
  final VoidCallback? onLanguageChanged;
  
  /// إذا كان true، سيتم استدعاء onLanguageChanged عند البناء الأولي
  final bool reloadOnInitialBuild;
}
```

### 🏗️ البنية الداخلية

#### 1. State Variables

```dart
class _LanguageAwareWidgetState extends State<LanguageAwareWidget> with RouteAware {
  String? _lastLanguageCode;              // آخر لغة معروفة
  bool _isScreenActive = true;            // هل الشاشة نشطة حالياً؟
  bool _languageChangedWhileInactive = false; // هل تغيرت اللغة بينما الشاشة غير نشطة؟
}
```

#### 2. دورة الحياة (Lifecycle)

##### `initState()`
- يحصل على اللغة الحالية من `LanguageCubit`
- يخزنها في `_lastLanguageCode`

##### `didChangeDependencies()`
- يسجل نفسه مع `LanguageRouteObserver` للاستماع لتغييرات المسار

##### `dispose()`
- يلغي تسجيل نفسه من `LanguageRouteObserver`

##### `build()`
- يستخدم `BlocListener<LanguageCubit>` للاستماع لتغييرات اللغة
- عند تغيير اللغة:
  - إذا كانت الشاشة نشطة (`_isScreenActive == true`): لا يستدعي callback (يترك `GlobalAppBar` يتعامل معه)
  - إذا كانت الشاشة غير نشطة (`_isScreenActive == false`): يضع علامة `_languageChangedWhileInactive = true`

#### 3. RouteAware Methods

##### `didPush()`
- يتم استدعاؤه عندما يتم دفع الشاشة الحالية إلى المكدس
- يضع `_isScreenActive = true`
- إذا تغيرت اللغة بينما كانت الشاشة غير نشطة، يستدعي `onLanguageChanged`

##### `didPopNext()`
- يتم استدعاؤه عندما يتم إغلاق شاشة أخرى والعودة إلى هذه الشاشة
- يضع `_isScreenActive = true`
- إذا تغيرت اللغة بينما كانت الشاشة غير نشطة، يستدعي `onLanguageChanged`

##### `didPop()`
- يتم استدعاؤه عندما يتم إغلاق الشاشة الحالية
- يضع `_isScreenActive = false`

##### `didPushNext()`
- يتم استدعاؤه عندما يتم الانتقال إلى شاشة أخرى
- يضع `_isScreenActive = false`

### 📊 مخطط تدفق البيانات

```
┌─────────────────────────────────────────────────────────────┐
│ LanguageAwareWidget                                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  BlocListener<LanguageCubit>                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ عند تغيير اللغة:                                      │  │
│  │                                                       │  │
│  │ if (_isScreenActive) {                               │  │
│  │   // الشاشة نشطة - لا تفعل شيئاً                    │  │
│  │   // GlobalAppBar سيتعامل مع التحديث                │  │
│  │ } else {                                             │  │
│  │   // الشاشة غير نشطة                                  │  │
│  │   _languageChangedWhileInactive = true               │  │
│  │ }                                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  RouteAware (didPopNext / didPush)                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ عند العودة إلى الشاشة:                                 │  │
│  │                                                       │  │
│  │ if (_languageChangedWhileInactive) {                  │  │
│  │   onLanguageChanged() // إعادة تحميل البيانات        │  │
│  │   _languageChangedWhileInactive = false               │  │
│  │ }                                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 💡 مثال الاستخدام

```dart
class BranchesScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final localizations = context.localizations;
    if (localizations == null) return const SizedBox.shrink();

    return LanguageAwareWidget(
      onLanguageChanged: () {
        // إعادة تحميل الفروع عند تغيير اللغة
        context.read<BranchesCubit>().loadBranches();
      },
      child: Scaffold(
        appBar: GlobalAppBar(
          showBackButton: true,
          title: localizations.translate('branches') ?? 'الفروع',
          onLanguageChanged: () {
            // تحديث فوري عند تغيير اللغة في هذه الشاشة
            context.read<BranchesCubit>().loadBranches();
          },
        ),
        body: BlocBuilder<BranchesCubit, BranchesState>(
          builder: (context, state) {
            // عرض الفروع...
          },
        ),
      ),
    );
  }
}
```

---

## GlobalAppBar

### 📝 الوصف

`GlobalAppBar` هو `StatelessWidget` يطبق `PreferredSizeWidget` ويوفر AppBar موحد لجميع الشاشات في التطبيق. يحتوي على مبدل اللغة الذي يستدعي callback عند تغيير اللغة مباشرة في الشاشة الحالية.

### 🔧 المعاملات (Parameters)

```dart
class GlobalAppBar extends StatelessWidget implements PreferredSizeWidget {
  /// Callback يتم استدعاؤه عند تغيير اللغة في الشاشة الحالية
  final VoidCallback? onLanguageChanged;
  
  /// عنوان الشاشة (يظهر بدلاً من رسالة الترحيب)
  final String? title;
  
  /// إذا كان true، يظهر زر الرجوع بدلاً من الشعار
  final bool showBackButton;
}
```

### 🏗️ البنية الداخلية

#### 1. المكونات الرئيسية

##### Leading (الجزء الأيسر)
- إذا `showBackButton == true`: يظهر `BackButton`
- إذا `showBackButton == false`: يظهر الشعار مع `GestureDetector` لفتح الـ Drawer

##### Title (العنوان)
- إذا `showBackButton == true && title != null`: يظهر العنوان
- خلاف ذلك: يظهر رسالة الترحيب (من `_buildWelcomeText`)

##### Actions (الأزرار)
- مبدل اللغة (`_buildLanguageSelector`)
- زر الإشعارات (`_buildNotificationButton`)

#### 2. مبدل اللغة (`_buildLanguageSelector`)

```dart
Widget _buildLanguageSelector(
  BuildContext context,
  LanguageState languageState,
  AppLocalizations localizations,
) {
  return InkWell(
    onTap: () async {
      // تغيير اللغة
      await context.read<LanguageCubit>().changeLanguage(
        languageState.locale.languageCode == 'ar' ? 'en' : 'ar',
      );
      
      // انتظار صغير لضمان تحديث LanguageService
      await Future.delayed(const Duration(milliseconds: 100));
      
      // استدعاء callback إذا كان موجوداً
      if (onLanguageChanged != null) {
        onLanguageChanged!();
      }
    },
    child: // UI للزر...
  );
}
```

**الخطوات:**
1. تغيير اللغة عبر `LanguageCubit.changeLanguage()`
2. انتظار 100ms لضمان تحديث `LanguageService`
3. استدعاء `onLanguageChanged` callback لإعادة تحميل البيانات

#### 3. رسالة الترحيب (`_buildWelcomeText`)

- يستخدم `StreamBuilder` للاستماع لحالة المصادقة
- يعرض:
  - إذا كان المستخدم مسجلاً: "صباح الخير / مساء الخير" + الاسم الأول
  - إذا لم يكن مسجلاً: "مرحباً بكم"

### 📊 مخطط تدفق البيانات

```
┌─────────────────────────────────────────────────────────────┐
│ GlobalAppBar                                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  مبدل اللغة (InkWell)                                      │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ عند الضغط:                                            │  │
│  │                                                       │  │
│  │ 1. LanguageCubit.changeLanguage()                    │  │
│  │ 2. انتظار 100ms                                       │  │
│  │ 3. onLanguageChanged() // إعادة تحميل البيانات      │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  BlocBuilder<LanguageCubit>                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ يعيد بناء AppBar عند تغيير اللغة                      │  │
│  │ (لتحديث النصوص والعناصر)                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 💡 مثال الاستخدام

```dart
// الشاشة الرئيسية (مع الشعار ورسالة الترحيب)
Scaffold(
  appBar: GlobalAppBar(
    onLanguageChanged: () {
      context.read<MainCubit>().loadData();
    },
  ),
  // ...
)

// الشاشة الفرعية (مع زر الرجوع والعنوان)
Scaffold(
  appBar: GlobalAppBar(
    showBackButton: true,
    title: localizations.translate('branches') ?? 'الفروع',
    onLanguageChanged: () {
      context.read<BranchesCubit>().loadBranches();
    },
  ),
  // ...
)
```

---

## LanguageRouteObserver

### 📝 الوصف

`LanguageRouteObserver` هو `RouteObserver<PageRoute<dynamic>>` يستخدم لتتبع حالة الشاشات (نشطة/غير نشطة). يتم تسجيله في `main.dart` كـ `NavigatorObserver`.

### 🔧 التسجيل في التطبيق

#### 1. في `dependency_injection.dart`:

```dart
// تسجيل LanguageRouteObserver كـ lazy singleton
getIt.registerLazySingleton<LanguageRouteObserver>(
  () => LanguageRouteObserver(),
);
```

#### 2. في `main.dart`:

```dart
MaterialApp(
  navigatorObservers: [
    GetIt.I.get<LanguageRouteObserver>(),
  ],
  // ...
)
```

### 🏗️ البنية الداخلية

```dart
class LanguageRouteObserver extends RouteObserver<PageRoute<dynamic>> {
  // هذا الـ observer يستخدمه LanguageAwareWidget
  // لاكتشاف متى تصبح الشاشات نشطة أو غير نشطة
}
```

**الوظيفة:**
- يتتبع جميع تغييرات المسار (push, pop, replace)
- يخبر `RouteAware` widgets (مثل `LanguageAwareWidget`) عندما:
  - يتم دفع شاشة جديدة (`didPush`)
  - يتم إغلاق شاشة (`didPop`)
  - يتم العودة إلى شاشة (`didPopNext`)
  - يتم الانتقال إلى شاشة أخرى (`didPushNext`)

---

## كيف يعمل النظام معاً

### 🔄 السيناريو الكامل

#### الحالة 1: تغيير اللغة في الشاشة الحالية

```
1. المستخدم في شاشة "الفروع"
2. المستخدم يضغط على مبدل اللغة في GlobalAppBar
3. GlobalAppBar.onLanguageChanged() يتم استدعاؤه
4. البيانات يتم إعادة تحميلها فوراً
5. LanguageAwareWidget يكتشف تغيير اللغة لكن لا يستدعي callback
   (لأن _isScreenActive == true)
```

**الكود:**

```dart
// في GlobalAppBar
onTap: () async {
  await context.read<LanguageCubit>().changeLanguage('en');
  await Future.delayed(const Duration(milliseconds: 100));
  if (onLanguageChanged != null) {
    onLanguageChanged!(); // ✅ يتم استدعاؤه
  }
}

// في LanguageAwareWidget
if (_isScreenActive) {
  // الشاشة نشطة - لا تفعل شيئاً
  // GlobalAppBar سيتعامل مع التحديث ✅
  _lastLanguageCode = currentLanguageCode;
}
```

#### الحالة 2: تغيير اللغة في شاشة أخرى ثم العودة

```
1. المستخدم في شاشة "الفروع" (اللغة: عربي)
2. المستخدم ينتقل إلى شاشة "تفاصيل الوثيقة"
   → LanguageAwareWidget.didPushNext() → _isScreenActive = false
3. المستخدم يغير اللغة إلى الإنجليزية في شاشة "تفاصيل الوثيقة"
   → LanguageAwareWidget يكتشف التغيير لكن _isScreenActive = false
   → _languageChangedWhileInactive = true
4. المستخدم يعود إلى شاشة "الفروع"
   → LanguageAwareWidget.didPopNext() → _isScreenActive = true
   → إذا _languageChangedWhileInactive == true:
      → onLanguageChanged() يتم استدعاؤه ✅
      → البيانات يتم إعادة تحميلها
```

**الكود:**

```dart
// في LanguageAwareWidget
// عند تغيير اللغة بينما الشاشة غير نشطة
if (!_isScreenActive) {
  _languageChangedWhileInactive = true; // ✅ وضع علامة
  _lastLanguageCode = currentLanguageCode;
}

// عند العودة إلى الشاشة
@override
void didPopNext() {
  _isScreenActive = true;
  if (_languageChangedWhileInactive && widget.onLanguageChanged != null) {
    _languageChangedWhileInactive = false;
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (mounted && widget.onLanguageChanged != null) {
        widget.onLanguageChanged!(); // ✅ يتم استدعاؤه
      }
    });
  }
}
```

### 📊 مخطط التدفق الكامل

```
┌─────────────────────────────────────────────────────────────────┐
│                    تغيير اللغة في الشاشة الحالية                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  المستخدم يضغط على مبدل اللغة                                   │
│         ↓                                                         │
│  GlobalAppBar._buildLanguageSelector()                          │
│         ↓                                                         │
│  LanguageCubit.changeLanguage()                                 │
│         ↓                                                         │
│  LanguageService.updateLanguage()                                │
│         ↓                                                         │
│  GlobalAppBar.onLanguageChanged() ✅                            │
│         ↓                                                         │
│  إعادة تحميل البيانات فوراً                                      │
│                                                                  │
│  LanguageAwareWidget يكتشف التغيير لكن:                          │
│  - _isScreenActive == true                                       │
│  - لا يستدعي callback (تجنب التكرار)                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              تغيير اللغة في شاشة أخرى ثم العودة                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  المستخدم في شاشة "أ"                                           │
│         ↓                                                         │
│  الانتقال إلى شاشة "ب"                                          │
│  → LanguageAwareWidget.didPushNext()                           │
│  → _isScreenActive = false                                       │
│         ↓                                                         │
│  تغيير اللغة في شاشة "ب"                                        │
│         ↓                                                         │
│  LanguageAwareWidget يكتشف التغيير:                              │
│  - _isScreenActive == false                                      │
│  - _languageChangedWhileInactive = true ✅                      │
│         ↓                                                         │
│  العودة إلى شاشة "أ"                                             │
│  → LanguageAwareWidget.didPopNext()                             │
│  → _isScreenActive = true                                        │
│  → إذا _languageChangedWhileInactive == true:                   │
│     → onLanguageChanged() ✅                                     │
│     → إعادة تحميل البيانات                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## أمثلة عملية من الكود

### مثال 1: شاشة الفروع (BranchesScreen)

```dart
class BranchesScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final localizations = context.localizations;
    if (localizations == null) return const SizedBox.shrink();

    return LanguageAwareWidget(
      onLanguageChanged: () {
        // إعادة تحميل الفروع عند العودة إلى الشاشة بعد تغيير اللغة
        context.read<BranchesCubit>().loadBranches();
      },
      child: Scaffold(
        appBar: GlobalAppBar(
          showBackButton: true,
          title: localizations.translate('branches') ?? 'الفروع',
          onLanguageChanged: () {
            // تحديث فوري عند تغيير اللغة في هذه الشاشة
            context.read<BranchesCubit>().loadBranches();
          },
        ),
        body: BlocBuilder<BranchesCubit, BranchesState>(
          builder: (context, state) {
            if (state is BranchesLoading) {
              return const Center(child: CircularProgressIndicator());
            } else if (state is BranchesLoaded) {
              return GridView.builder(
                // عرض الفروع...
              );
            } else if (state is BranchesError) {
              return Center(child: Text(state.message));
            }
            return const SizedBox.shrink();
          },
        ),
      ),
    );
  }
}
```

### مثال 2: شاشة تفاصيل الوثيقة (DocumentDetailsScreen)

```dart
class DocumentDetailsScreen extends StatelessWidget {
  final String documentNumber;

  const DocumentDetailsScreen({
    super.key,
    required this.documentNumber,
  });

  @override
  Widget build(BuildContext context) {
    final localizations = context.localizations;
    if (localizations == null) return const SizedBox.shrink();

    return LanguageAwareWidget(
      onLanguageChanged: () {
        // إعادة تحميل تفاصيل الوثيقة عند العودة إلى الشاشة
        context.read<DocumentDetailsCubit>().loadDocumentDetails(documentNumber);
      },
      child: Scaffold(
        appBar: GlobalAppBar(
          showBackButton: true,
          title: localizations.translate('document_details') ?? 'تفاصيل الوثيقة',
          onLanguageChanged: () {
            // تحديث فوري عند تغيير اللغة في هذه الشاشة
            context.read<DocumentDetailsCubit>().loadDocumentDetails(documentNumber);
          },
        ),
        body: BlocBuilder<DocumentDetailsCubit, DocumentDetailsState>(
          builder: (context, state) {
            if (state is DocumentDetailsLoading) {
              return const Center(child: CircularProgressIndicator());
            } else if (state is DocumentDetailsLoaded) {
              return SingleChildScrollView(
                // عرض تفاصيل الوثيقة...
              );
            } else if (state is DocumentDetailsError) {
              return Center(child: Text(state.message));
            }
            return const SizedBox.shrink();
          },
        ),
      ),
    );
  }
}
```

### مثال 3: شاشة رئيسية (MainBidderScreen)

```dart
class MainBidderScreen extends StatelessWidget {
  const MainBidderScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final localizations = context.localizations;
    if (localizations == null) return const SizedBox.shrink();

    return LanguageAwareWidget(
      onLanguageChanged: () {
        // إعادة تحميل البيانات عند العودة إلى الشاشة
        if (GetIt.I.isRegistered<BidderSharedCubit>()) {
          GetIt.I.get<BidderSharedCubit>().loadCategoriesAndClassifications();
        }
        context.read<MainBidderCubit>().loadBidders();
      },
      child: Scaffold(
        appBar: GlobalAppBar(
          // لا showBackButton (الشاشة الرئيسية)
          // لا title (يظهر رسالة الترحيب)
          onLanguageChanged: () {
            // تحديث فوري عند تغيير اللغة في هذه الشاشة
            if (GetIt.I.isRegistered<BidderSharedCubit>()) {
              GetIt.I.get<BidderSharedCubit>().loadCategoriesAndClassifications();
            }
            context.read<MainBidderCubit>().loadBidders();
          },
        ),
        drawer: AppDrawer(/* ... */),
        body: BlocBuilder<MainBidderCubit, MainBidderState>(
          // عرض البيانات...
        ),
      ),
    );
  }
}
```

### مثال 4: شاشة تفاصيل مع timestamp (InventoryDetailsScreen)

```dart
class InventoryDetailsScreen extends StatelessWidget {
  final int itemId;

  const InventoryDetailsScreen({
    super.key,
    required this.itemId,
  });

  @override
  Widget build(BuildContext context) {
    final localizations = context.localizations;
    if (localizations == null) return const SizedBox.shrink();

    return LanguageAwareWidget(
      onLanguageChanged: () {
        // إعادة تحميل التفاصيل عند العودة إلى الشاشة
        context.read<InventoryDetailsCubit>().loadDetails(itemId);
      },
      child: Scaffold(
        appBar: GlobalAppBar(
          showBackButton: true,
          title: localizations.translate('inventory_details') ?? 'تفاصيل المخزون',
          onLanguageChanged: () {
            // تحديث فوري عند تغيير اللغة في هذه الشاشة
            context.read<InventoryDetailsCubit>().loadDetails(itemId);
          },
        ),
        body: BlocBuilder<InventoryDetailsCubit, InventoryDetailsState>(
          builder: (context, state) {
            if (state is InventoryDetailsLoaded) {
              // استخدام ValueKey مع timestamp لإجبار إعادة البناء
              return ValueKey(
                '${itemId}_${state.timestamp}_${localizations.locale.languageCode}',
                child: _buildContent(state.details),
              );
            }
            // ...
          },
        ),
      ),
    );
  }
}
```

**ملاحظة:** في شاشات التفاصيل، قد نحتاج إلى استخدام `ValueKey` مع `timestamp` لإجبار `BlocBuilder` على إعادة بناء الـ widget حتى لو لم يتغير الكائن `details` نفسه.

---

## سيناريوهات الاستخدام

### السيناريو 1: تغيير اللغة في الشاشة الحالية

**الخطوات:**
1. المستخدم في شاشة "الفروع"
2. المستخدم يضغط على مبدل اللغة في `GlobalAppBar`
3. `GlobalAppBar.onLanguageChanged()` يتم استدعاؤه
4. `BranchesCubit.loadBranches()` يتم استدعاؤه
5. البيانات يتم إعادة تحميلها باللغة الجديدة
6. `LanguageAwareWidget` يكتشف التغيير لكن لا يستدعي callback (تجنب التكرار)

**النتيجة:** ✅ تحديث فوري

### السيناريو 2: تغيير اللغة في شاشة أخرى ثم العودة

**الخطوات:**
1. المستخدم في شاشة "الفروع" (اللغة: عربي)
2. المستخدم ينتقل إلى شاشة "تفاصيل الوثيقة"
3. المستخدم يغير اللغة إلى الإنجليزية في شاشة "تفاصيل الوثيقة"
4. `LanguageAwareWidget` في شاشة "الفروع" يكتشف التغيير ويضع `_languageChangedWhileInactive = true`
5. المستخدم يعود إلى شاشة "الفروع"
6. `LanguageAwareWidget.didPopNext()` يتم استدعاؤه
7. `onLanguageChanged()` يتم استدعاؤه
8. `BranchesCubit.loadBranches()` يتم استدعاؤه
9. البيانات يتم إعادة تحميلها باللغة الإنجليزية

**النتيجة:** ✅ تحديث تلقائي عند العودة

### السيناريو 3: تغيير اللغة عدة مرات في شاشات مختلفة

**الخطوات:**
1. المستخدم في شاشة "أ" (اللغة: عربي)
2. المستخدم ينتقل إلى شاشة "ب"
3. المستخدم يغير اللغة إلى الإنجليزية في شاشة "ب"
4. المستخدم ينتقل إلى شاشة "ج"
5. المستخدم يغير اللغة إلى الفرنسية في شاشة "ج"
6. المستخدم يعود إلى شاشة "ب"
7. شاشة "ب" تتحدث إلى الفرنسية تلقائياً
8. المستخدم يعود إلى شاشة "أ"
9. شاشة "أ" تتحدث إلى الفرنسية تلقائياً

**النتيجة:** ✅ جميع الشاشات تتحدث إلى آخر لغة تم اختيارها

---

## أفضل الممارسات

### ✅ افعل (Do)

1. **استخدم `LanguageAwareWidget` في جميع الشاشات التي تعرض بيانات من API**
   ```dart
   return LanguageAwareWidget(
     onLanguageChanged: () {
       context.read<YourCubit>().loadData();
     },
     child: Scaffold(/* ... */),
   );
   ```

2. **استخدم `onLanguageChanged` في `GlobalAppBar`**
   ```dart
   appBar: GlobalAppBar(
     onLanguageChanged: () {
       context.read<YourCubit>().loadData();
     },
   ),
   ```

3. **استخدم `ValueKey` مع `timestamp` في شاشات التفاصيل**
   ```dart
   ValueKey(
     '${itemId}_${state.timestamp}_${localizations.locale.languageCode}',
     child: _buildContent(state.data),
   ),
   ```

4. **أضف `timestamp` إلى `Loaded` states في Cubits**
   ```dart
   class YourLoadedState extends YourState {
     final YourData data;
     final int timestamp; // ✅ أضف هذا
    
     YourLoadedState({
       required this.data,
       this.timestamp = 0,
     });
    
     @override
     List<Object?> get props => [data, timestamp]; // ✅ أضف timestamp
   }
   ```

### ❌ لا تفعل (Don't)

1. **لا تستدعي `loadData()` مباشرة في `build()`**
   ```dart
   // ❌ خطأ
   @override
   Widget build(BuildContext context) {
     context.read<YourCubit>().loadData(); // ❌ سيتم استدعاؤه في كل rebuild
     return Scaffold(/* ... */);
   }
   ```

2. **لا تنسَ إضافة `timestamp` إلى `Loaded` states**
   ```dart
   // ❌ خطأ
   class YourLoadedState extends YourState {
     final YourData data;
     // ❌ timestamp مفقود
   }
   ```

3. **لا تستخدم `LanguageAwareWidget` بدون `onLanguageChanged`**
   ```dart
   // ❌ خطأ (ما لم تكن متأكداً أنك لا تحتاج إلى تحديث)
   return LanguageAwareWidget(
     child: Scaffold(/* ... */),
   );
   ```

---

## الخلاصة

نظام إدارة اللغة والتحديث التلقائي للبيانات يضمن:

1. ✅ **تحديث فوري** عند تغيير اللغة في الشاشة الحالية (عبر `GlobalAppBar`)
2. ✅ **تحديث تلقائي** عند العودة إلى شاشة بعد تغيير اللغة في شاشة أخرى (عبر `LanguageAwareWidget`)
3. ✅ **عدم التكرار** (تجنب استدعاء callback مرتين)
4. ✅ **سهولة الاستخدام** (wrapper widget بسيط)
5. ✅ **مرونة** (يعمل مع أي Cubit/Bloc)

**الملفات الرئيسية:**
- `lib/core/widgets/language_aware_widget.dart`
- `lib/app_modules/global_app_bar.dart`
- `lib/core/navigation/language_route_observer.dart`

**الشاشات المطبقة:**
- ✅ جميع شاشات قسم العملاء (Customers)
- ✅ جميع شاشات قسم المناقصات (Bidders)
- ✅ جميع الشاشات التي تعرض بيانات من API

---

**آخر تحديث:** 2024

