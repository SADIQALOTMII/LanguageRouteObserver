# Language Aware Widget - دليل الاستخدام

## نظرة عامة

`LanguageAwareWidget` هو widget wrapper يستمع تلقائياً لتغييرات اللغة ويستدعي callback لإعادة تحميل البيانات عند تغيير اللغة.

## المشكلة التي يحلها

عندما يغير المستخدم اللغة في شاشة معينة (مثل شاشة تفاصيل الوثيقة) ثم يعود إلى الشاشة السابقة (مثل شاشة الفروع)، يجب أن تتحدث الشاشة السابقة تلقائياً لتعكس اللغة الجديدة.

## كيفية الاستخدام

### 1. استيراد Widget

```dart
import '../../../../../core/widgets/language_aware_widget.dart';
```

### 2. استخدام Widget في الشاشة

```dart
class BranchesScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final localizations = context.localizations;
    if (localizations == null) return const SizedBox.shrink();

    return LanguageAwareWidget(
      onLanguageChanged: () {
        // إعادة تحميل البيانات عند تغيير اللغة
        context.read<BranchesCubit>().loadBranches();
      },
      child: Scaffold(
        appBar: GlobalAppBar(
          showBackButton: true,
          title: localizations.translate('branches') ?? 'الفروع',
          onLanguageChanged: () {
            // هذا callback يتم استدعاؤه عند تغيير اللغة مباشرة في هذه الشاشة
            context.read<BranchesCubit>().loadBranches();
          },
        ),
        // باقي الشاشة...
      ),
    );
  }
}
```

### 3. المعاملات

- `child` (مطلوب): Widget الذي تريد تغليفه
- `onLanguageChanged` (اختياري): Callback يتم استدعاؤه عند تغيير اللغة
- `reloadOnInitialBuild` (اختياري، افتراضي: `false`): إذا كان `true`، سيتم استدعاء `onLanguageChanged` عند البناء الأولي

## كيف يعمل

1. `LanguageAwareWidget` يستخدم `BlocListener` للاستماع لتغييرات `LanguageCubit`
2. عندما تتغير اللغة، يتم مقارنة اللغة الحالية باللغة السابقة
3. إذا تغيرت اللغة، يتم استدعاء `onLanguageChanged` callback
4. هذا يضمن أن الشاشة ستتحدث تلقائياً حتى لو تغيرت اللغة في شاشة أخرى

## مثال كامل

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
        // إعادة تحميل تفاصيل الوثيقة عند تغيير اللغة
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
          // ...
        ),
      ),
    );
  }
}
```

## ملاحظات مهمة

1. **استخدم `onLanguageChanged` في `GlobalAppBar`**: هذا يضمن التحديث الفوري عند تغيير اللغة مباشرة في الشاشة الحالية
2. **استخدم `LanguageAwareWidget`**: هذا يضمن التحديث التلقائي عند العودة إلى الشاشة بعد تغيير اللغة في شاشة أخرى
3. **لا حاجة لتخزين اللغة يدوياً**: `LanguageAwareWidget` يتتبع اللغة تلقائياً

## الشاشات الموصى بتطبيقها

- ✅ `BranchesScreen`
- ✅ `DocumentDetailsScreen`
- ✅ `FamilyScreen`
- ✅ `FamilyInsuranceDocumentsScreen`
- ✅ `FamilyDocumentDetailsScreen`
- ✅ جميع الشاشات التي تعرض بيانات من API

