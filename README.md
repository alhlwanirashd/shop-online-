.nojekyll
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>منصة التاجر - تسجيل دخول مندوب Firebase</title>

  <!-- Firebase SDK -->
  <script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
  <script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-auth-compat.js"></script>

  <script src="https://maps.googleapis.com/maps/api/js?key=AIzaSyBvnWpykz1k45L6LM0A1ekIXAg3j1EMzls&libraries=places"></script>

  <style>
    body {
      margin: 0;
      font-family: 'Cairo', sans-serif;
      background-color: #f9f9f9;
    }
    header {
      background-color: #ff5722;
      padding: 1rem 2rem;
      color: white;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    header h1 {
      margin: 0;
    }
    .container {
      padding: 2rem;
      max-width: 1000px;
      margin: auto;
    }
    .form-section, .product-list, .vendor-table, .login-section, .commission-section, .delivery-login-section, .map-section, .payment-section {
      margin-bottom: 2rem;
      background: white;
      padding: 1.5rem;
      border-radius: 12px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.05);
    }
    input, button, select {
      padding: 0.5rem;
      margin-top: 0.5rem;
      width: 100%;
      border: 1px solid #ccc;
      border-radius: 8px;
      box-sizing: border-box;
    }
    button {
      background-color: #ff5722;
      color: white;
      border: none;
      cursor: pointer;
    }
    button:hover {
      background-color: #e64a19;
    }
    .product {
      padding: 1rem;
      border-bottom: 1px solid #eee;
    }
    table {
      width: 100%;
      border-collapse: collapse;
    }
    th, td {
      padding: 0.75rem;
      border: 1px solid #eee;
      text-align: center;
    }
    th {
      background: #f3f4f6;
    }
    #map {
      width: 100%;
      height: 300px;
      margin-top: 1rem;
    }
    #currentDeliveryInfo {
      margin-bottom: 1rem;
      font-weight: bold;
      color: #333;
    }
  </style>
</head>
<body>
  <header>
    <h1>منصة التاجر</h1>
  </header>

  <div class="container">

    <!-- نموذج إضافة المنتج -->
    <section class="form-section">
      <h2>إضافة منتج جديد</h2>
      <input type="text" id="productName" placeholder="اسم المنتج" />
      <input type="number" id="productPrice" placeholder="السعر" />
      <select id="productType">
        <option value="طعام">طعام</option>
        <option value="ملابس">ملابس</option>
        <option value="إلكترونيات">إلكترونيات</option>
      </select>
      <button onclick="addProduct()">إضافة</button>
    </section>

    <!-- قائمة المنتجات -->
    <section class="product-list">
      <h2>المنتجات</h2>
      <div id="productContainer"></div>
    </section>

    <!-- سجل التوصيلات -->
    <section class="delivery-log">
      <h2>سجل التوصيلات</h2>
      <ul id="deliveryHistory"></ul>
    </section>

    <!-- خريطة واحتساب عمولة -->
    <section class="map-section">
      <h2>احتساب عمولة التوصيل</h2>
      <div id="currentDeliveryInfo">المندوب الحالي: لم يتم تسجيل الدخول</div>
      <input type="text" id="origin" placeholder="موقع الانطلاق" />
      <input type="text" id="destination" placeholder="الوجهة" />
      <button onclick="calculateDistance()">احسب المسافة</button>
      <p id="distanceOutput"></p>
      <div id="map"></div>
    </section>

    <!-- تسجيل دخول مندوب برقم الهاتف -->
    <section class="delivery-login-section">
      <h2>تسجيل دخول المندوب برقم الهاتف</h2>
      <input type="tel" id="deliveryPhone" placeholder="أدخل رقم الهاتف (مثلاً +9627xxxxxxx)" />
      <div id="recaptcha-container"></div>
      <button onclick="sendOTP()">إرسال رمز التحقق</button>

      <input type="text" id="otp" placeholder="أدخل رمز التحقق" style="margin-top: 10px;" />
      <button onclick="verifyOTP()">تأكيد الدخول</button>
    </section>

    <!-- قسم الدفع -->
    <section class="payment-section">
      <h2>الدفع الإلكتروني عبر كليك</h2>
      <p>اختر قيمة للدفع:</p>
      <input type="number" id="paymentAmount" placeholder="المبلغ بالدينار الأردني" />
      <button onclick="payWithCliq()">ادفع عبر كليك</button>
    </section>
  </div>

  <script>
    // Firebase إعدادات - استبدل القيم بقيم مشروعك من Firebase Console
    const firebaseConfig = {
      apiKey: "YOUR_API_KEY_HERE",
      authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
      projectId: "YOUR_PROJECT_ID",
      storageBucket: "YOUR_PROJECT_ID.appspot.com",
      messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
      appId: "YOUR_APP_ID"
    };

    firebase.initializeApp(firebaseConfig);
    const auth = firebase.auth();

    // متغيرات النظام
    let commissionRate = 0.10;
    const products = [];
    const vendorSales = {};
    let currentVendor = 'تاجر_تجريبي'; // يمكنك تحديث النظام لإضافة تسجيل تجار لاحقًا
    let currentDelivery = '';
    const deliveryLogs = [];
    let confirmationResult;

    // تسجيل دخول مندوب Firebase
    function sendOTP() {
      const phoneNumber = document.getElementById('deliveryPhone').value.trim();
      if (!phoneNumber) {
        alert('يرجى إدخال رقم الهاتف');
        return;
      }

      window.recaptchaVerifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', {
        size: 'invisible',
        callback: (response) => {
          console.log('reCAPTCHA تم التحقق');
        }
      });

      auth.signInWithPhoneNumber(phoneNumber, window.recaptchaVerifier)
        .then((result) => {
          confirmationResult = result;
          alert('تم إرسال رمز التحقق إلى هاتفك');
        })
        .catch((error) => {
          alert('حدث خطأ أثناء إرسال رمز التحقق: ' + error.message);
        });
    }

    function verifyOTP() {
      const code = document.getElementById('otp').value.trim();
      if (!code) {
        alert('يرجى إدخال رمز التحقق');
        return;
      }

      confirmationResult.confirm(code)
        .then((result) => {
          const user = result.user;
          currentDelivery = user.phoneNumber;
          document.getElementById("currentDeliveryInfo").innerText = `المندوب الحالي: ${currentDelivery}`;
          alert('تم تسجيل الدخول بنجاح!');
        })
        .catch((error) => {
          alert('رمز التحقق غير صحيح أو انتهت صلاحيته');
        });
    }

    // إضافة منتج جديد
    function addProduct() {
      const name = document.getElementById('productName').value.trim();
      const price = parseFloat(document.getElementById('productPrice').value);
      const type = document.getElementById('productType').value;

      if (!name || isNaN(price)) {
        alert('يرجى تعبئة جميع الحقول بشكل صحيح');
        return;
      }

      if (!currentVendor) {
        alert('يجب تسجيل الدخول كتاجر أولاً (الوظيفة مؤقتة حالياً)');
        return;
      }

      const product = { name, price, type, vendor: currentVendor };
      products.push(product);

      if (!vendorSales[currentVendor]) {
        vendorSales[currentVendor] = 0;
      }
      vendorSales[currentVendor] += price;

      renderProducts();
      alert('تم إضافة المنتج بنجاح');
      document.getElementById('productName').value = '';
      document.getElementById('productPrice').value = '';
    }

    // عرض المنتجات
    function renderProducts() {
      const container = document.getElementById('productContainer');
      container.innerHTML = '';
      products.forEach(p => {
        const div = document.createElement('div');
        div.className = 'product';
        div.textContent = `${p.name} (${p.type}) - ${p.price.toFixed(2)} د.ا (بواسطة ${p.vendor})`;
        container.appendChild(div);
      });
    }

    // احتساب المسافة وعمولة التوصيل
    function calculateDistance() {
      const origin = document.getElementById("origin").value.trim();
      const destination = document.getElementById("destination").value.trim();
      if (!origin || !destination) {
        alert('يرجى إدخال موقع الانطلاق والوجهة');
        return;
      }

      const service = new google.maps.DistanceMatrixService();
      service.getDistanceMatrix(
        {
          origins: [origin],
          destinations: [destination],
          travelMode: google.maps.TravelMode.DRIVING
