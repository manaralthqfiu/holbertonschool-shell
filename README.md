# Part 2: Implementation of Business Logic and API Endpoints

## مقدمة

في هذا الجزء من مشروع HBnB نبدأ مرحلة التنفيذ الفعلي بناءً على التصميم الذي تم إنشاؤه في Part 1.

في Part 1 قمنا بتصميم الـ UML Class Diagram للـ Business Logic Layer،  
وفي Part 2 نقوم بتحويل هذا التصميم إلى كود فعلي باستخدام Python و Flask.

في هذه المرحلة نركز على:

- بناء طبقة الـ Business Logic
- إنشاء طبقة الـ Presentation (API Endpoints)
- تطبيق مبادئ التصميم النظيف Modular Design
- استخدام flask و flask-restx لبناء RESTful API
- تنفيذ عمليات CRUD الأساسية

في هذه المرحلة لا نقوم بتنفيذ:
- JWT Authentication
- Role-based Access Control
- Database حقيقية

الهدف هو بناء أساس قوي وقابل للتوسعة في الأجزاء القادمة.

---

## الهيكلة العامة للمشروع

داخل مجلد part2 تم تنظيم المشروع بشكل modular كما يلي:

```
part2/
│
├── app/
│   ├── models/
│   │   ├── base_model.py
│   │   ├── user.py
│   │   ├── place.py
│   │   ├── review.py
│   │   └── amenity.py
│
├── facade.py
├── storage.py
├── app.py
```

طبقة models تمثل Business Logic Layer.  
طبقة app.py تمثل Presentation Layer.

---

## Business Logic Layer

### 1) BaseModel

تم إنشاء BaseModel لتجنب تكرار الخصائص المشتركة بين جميع الكلاسات.

كل كيان في النظام يحتاج إلى:
- id
- created_at
- updated_at

بدل تكرارها في كل كلاس، قمنا باستخدام Inheritance.

الكود:

```python
import uuid
from datetime import datetime


class BaseModel:
    def __init__(self):
        self.id = str(uuid.uuid4())
        self.created_at = datetime.utcnow()
        self.updated_at = datetime.utcnow()

    def save(self):
        self.updated_at = datetime.utcnow()
```

شرح:

- uuid.uuid4() يولد معرف فريد لكل كائن.
- datetime.utcnow() يسجل وقت الإنشاء.
- save() تستخدم لتحديث وقت آخر تعديل.

استخدام UUID أفضل من الأرقام المتسلسلة لأنه:
- فريد عالميًا
- يصعب تخمينه
- مناسب للمشاريع الكبيرة والقابلة للتوسع

---

### 2) User Class

يمثل المستخدم داخل النظام.

الكود:

```python
from .base_model import BaseModel


class User(BaseModel):
    def __init__(self, first_name, last_name, email, password):
        super().__init__()

        if not first_name or not last_name:
            raise ValueError("First and last name required")

        if "@" not in email:
            raise ValueError("Invalid email")

        self.first_name = first_name
        self.last_name = last_name
        self.email = email
        self.password = password

        self.places = []
        self.reviews = []
```

شرح:

- يتم التحقق من صحة البيانات (Validation).
- المستخدم يمكنه امتلاك عدة Places.
- المستخدم يمكنه كتابة عدة Reviews.

Validation مهم لضمان سلامة البيانات ومنع إدخال قيم غير صحيحة.

---

### 3) Place Class

يمثل المكان الذي يتم عرضه في النظام.

الكود:

```python
from .base_model import BaseModel


class Place(BaseModel):
    def __init__(self, title, description, price, owner):
        super().__init__()

        if price < 0:
            raise ValueError("Price must be positive")

        self.title = title
        self.description = description
        self.price = price
        self.owner = owner

        self.reviews = []
        self.amenities = []

        owner.places.append(self)
```

شرح:

- يتم التأكد أن السعر ليس سالبًا.
- كل Place مرتبط بمستخدم Owner.
- يتم إضافة المكان تلقائيًا لقائمة أماكن المستخدم للحفاظ على تكامل العلاقة.

---

### 4) Review Class

تمثل التقييم المرتبط بمستخدم ومكان.

الكود:

```python
from .base_model import BaseModel


class Review(BaseModel):
    def __init__(self, text, rating, user, place):
        super().__init__()

        if rating < 1 or rating > 5:
            raise ValueError("Rating must be between 1 and 5")

        self.text = text
        self.rating = rating
        self.user = user
        self.place = place

        user.reviews.append(self)
        place.reviews.append(self)
```

شرح:

- التقييم يجب أن يكون بين 1 و 5.
- كل Review مرتبط بمستخدم ومكان.
- يتم إضافته للطرفين للحفاظ على تكامل البيانات.

---

### 5) Amenity Class

يمثل الخدمات المتوفرة في المكان.

```python
class Amenity(BaseModel):
    def __init__(self, name, description):
        super().__init__()

        self.name = name
        self.description = description
```

---

## لماذا استخدمنا Inheritance؟

لأن جميع الكلاسات تحتاج نفس الخصائص الأساسية.  
استخدام BaseModel يمنع تكرار الكود ويحقق مبدأ DRY (Don’t Repeat Yourself).

هذا يحسن قابلية الصيانة ويسهل التوسعة مستقبلاً.

---

## Storage المؤقت

بما أننا لم نستخدم Database في هذه المرحلة، تم إنشاء تخزين مؤقت باستخدام Dictionary.

storage.py:

```python
users = {}
```

عند إنشاء مستخدم:

```python
users[user.id] = user
```

---

## Facade Pattern

تم إنشاء Facade لعزل طبقة الـ Presentation عن Business Logic.

facade.py:

```python
from models.user import User
from storage import users

class HBnBFacade:

    @staticmethod
    def create_user(data):
        user = User(
            data["first_name"],
            data["last_name"],
            data["email"],
            data["password"]
        )

        users[user.id] = user
        return user
```

الهدف من Facade:
- منع الـ Endpoint من التعامل مباشرة مع الكلاسات
- تنظيم الكود
- تحقيق Separation of Concerns

---

## Presentation Layer (Flask Endpoint)

app.py:

```python
from flask import Flask, request, jsonify
from facade import HBnBFacade

app = Flask(__name__)

@app.route("/users", methods=["POST"])
def create_user():
    data = request.get_json()

    try:
        user = HBnBFacade.create_user(data)

        return jsonify({
            "id": user.id,
            "first_name": user.first_name,
            "last_name": user.last_name,
            "email": user.email
        }), 201

    except ValueError as e:
        return jsonify({"error": str(e)}), 400
```

شرح تدفق العملية:

1. يستقبل الـ endpoint طلب POST.
2. يتم استخراج البيانات من JSON.
3. يتم إرسال البيانات إلى Facade.
4. يتم إنشاء User وحفظه في storage.
5. يتم إرجاع البيانات بدون password.
6. يتم استخدام status code 201 عند الإنشاء.

عدم إرجاع password لأسباب أمنية.

---

## اختبار الـ API

يتم تشغيل السيرفر باستخدام:

```
flask run
```

ثم إرسال طلب POST إلى:

```
http://localhost:5000/users
```

باستخدام Postman أو cURL.

---

## الخلاصة

في هذا الجزء تم:

- تحويل التصميم إلى كود فعلي.
- بناء Business Logic Layer باستخدام OOP.
- تطبيق Inheritance و Validation.
- استخدام UUID كمعرف فريد.
- تنفيذ Facade Pattern.
- إنشاء أول API Endpoint.
- اختبار العمليات الأساسية.

بهذا يكون الأساس جاهز للانتقال إلى بقية المهام في Part 2 ثم إضافة المصادقة في 

1️⃣ Class (الكلاس)
--------------------------------------------------
أنتِ تكتبين القالب فقط، ما فيه بيانات حقيقية
مثال: place.py
class Place:
    def __init__(self, title, description, price, latitude, longitude, owner):
        self.title = title
        self.description = description
        self.price = price
        self.latitude = latitude
        self.longitude = longitude
        self.owner_id = owner.id
        self.reviews = []
        self.amenities = []

-----------------------------------------------
2️⃣ Object حي (في الذاكرة)
--------------------------------------------------
نوره أو الـ API تحول بيانات المستخدم (أو بيانات اختبار) لكائن حي
مثال:
owner = User("Manar", "Talal", "manar@email.com")
place = Place("Cozy Apartment", "Nice place to stay", 100, 37.77, -122.41, owner)

- البيانات موجودة مؤقتًا في الذاكرة
- تستخدمها البرنامج مباشرة
- لو قفلنا البرنامج → كل شيء يروح

-----------------------------------------------
3️⃣ JSON مؤقت (اختياري للعرض)
--------------------------------------------------
يمكن نرجع البيانات كـ JSON للمستخدم من Object
مثال:
{
  "title": "Cozy Apartment",
  "description": "Nice place to stay",
  "price": 100,
  "latitude": 37.77,
  "longitude": -122.41,
  "owner_id": "uuid-of-user"
}

- ما ينحفظ في أي ملف تلقائي
- فقط للعرض أو اختبار الـ API

-----------------------------------------------
4️⃣ JSON دائم (لاحقًا في Part 3)
--------------------------------------------------
بعد إضافة Persistence Layer (قاعدة بيانات أو JSON file)
- Objects تُخزن على القرص
- البيانات تبقى محفوظة بعد إغلاق البرنامج
- يمكن لأي حد فتح الملف ويشوف بيانات المستخدمين
