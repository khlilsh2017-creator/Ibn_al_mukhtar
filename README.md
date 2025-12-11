<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>إتمام الطلب</title>
<link href="https://fonts.googleapis.com/css2?family=Tajawal:wght@400;500;700;800&display=swap" rel="stylesheet">
<style>
  :root {
    --aliexpress-red: #FF4747; 
    --bg-color: #f2f2f2;
    --card-bg: #ffffff;
    --text-color: #222;
  }
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  body {
    font-family: 'Tajawal', sans-serif; margin: 0; background: var(--bg-color); padding-bottom: 50px;
  }
  header {
    background: #fff; padding: 15px; text-align: center; border-bottom: 1px solid #eee;
    font-size: 18px; font-weight: 700; color: var(--aliexpress-red);
  }
  .container { padding: 15px; max-width: 600px; margin: 0 auto; }
  
  /* نموذج إدخال البيانات */
  .info-form {
    background: var(--card-bg); padding: 20px; border-radius: 12px; margin-bottom: 20px;
    box-shadow: 0 2px 10px rgba(0,0,0,0.03);
  }
  label { display: block; margin-bottom: 5px; font-size: 14px; font-weight: 600; color: #555; }
  input, textarea {
    width: 100%; padding: 12px; margin-bottom: 15px; border: 1px solid #ddd;
    border-radius: 8px; font-family: inherit; font-size: 14px;
  }
  textarea { resize: vertical; }

  /* ملخص الطلب */
  .summary-card {
    background: #fff; padding: 15px; border-radius: 12px;
    border: 1px solid #eee; margin-top: 15px;
  }
  .summary-row { display: flex; justify-content: space-between; margin-bottom: 8px; font-size: 14px; }
  .total-row { border-top: 2px dashed #eee; padding-top: 10px; font-weight: 700; font-size: 16px; }
  .final-price { color: var(--aliexpress-red); font-size: 20px; font-weight: 800; }

  /* زر الإرسال */
  .submit-btn {
    width: 100%; padding: 15px; margin-top: 25px; border: none;
    background: linear-gradient(90deg, #FF8238, #FF4747);
    color: #fff; font-size: 16px; font-weight: bold; border-radius: 99px;
    box-shadow: 0 4px 10px rgba(255, 71, 71, 0.4);
    cursor: pointer;
  }
  .submit-btn:disabled { opacity: 0.7; cursor: not-allowed; }
  
  .cart-item { margin-bottom: 10px; border-bottom: 1px dotted #eee; padding-bottom: 5px; }
  .cart-item span { font-weight: 600; }
  
  .error-message { color: var(--aliexpress-red); text-align: center; margin-top: 20px; font-weight: bold; }
</style>
</head>
<body>

<header>🛒 إتمام الطلب</header>

<div class="container">
  
  <div class="info-form">
    <h3>بيانات التوصيل</h3>
    <label for="clientName">الاسم الكامل:</label>
    <input type="text" id="clientName" placeholder="الاسم المطلوب في الطلب" required>

    <label for="clientPhone">رقم الجوال:</label>
    <input type="tel" id="clientPhone" placeholder="967XXXXXXXX" required>

    <label for="clientLocation">العنوان التفصيلي (المدينة والحي):</label>
    <textarea id="clientLocation" placeholder="مثال: صنعاء - حي الأصبحي - جوار مدرسة كذا" required></textarea>
  </div>

  <div class="summary-card">
    <h3>ملخص الفاتورة</h3>
    <div id="orderItemsSummary"></div>
    
    <div class="summary-row">
        <span>رسوم التوصيل:</span>
        <span id="shippingSummary">0 ر.ي</span>
    </div>

    <div class="total-row">
        <span>المجموع النهائي:</span>
        <span class="final-price" id="finalTotalSummary">0 ر.ي</span>
    </div>
  </div>

  <button id="submitOrderBtn" class="submit-btn" onclick="submitOrder()">تأكيد الطلب الآن</button>
  <div id="errorMessage" class="error-message" style="display:none;"></div>
  
  <div id="successMessage" class="error-message" style="color:green; display:none;">
    ✅ تم استلام طلبك بنجاح! سيتم التواصل معك قريباً.
    <br><a href="index.html" style="color:var(--aliexpress-red); text-decoration:none;">العودة للمتجر</a>
  </div>
</div>

<script type="module">
  // 🔥🔥🔥 بيانات Firebase (يجب نسخها من index.html) 🔥🔥🔥
  import { initializeApp } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-app.js";
  import { getFirestore, collection, addDoc } from "https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore.js";

  const firebaseConfig = {
    apiKey: "AIzaSyB5mEBB0plzw_d-gAth-TK0xR523IX6nGg",
    authDomain: "my-fils-575be.firebaseapp.com",
    projectId: "my-fils-575be",
    storageBucket: "my-fils-575be.firebasestorage.app",
    messagingSenderId: "670428095338",
    appId: "1:670428095338:web:75d4a577cc648c208d7b13"
  };

  const app = initializeApp(firebaseConfig);
  const db = getFirestore(app);
  // 🔥 جدول جديد لحفظ الطلبات
  const ordersCol = collection(db, "orders"); 

  let currentCart = [];
  let finalTotalAmount = 0;
  let shippingDetails = {};

  // ------------------------------------------------------------------
  // 1. عرض الملخص عند تحميل الصفحة
  // ------------------------------------------------------------------
  function loadCartSummary() {
    const cartData = JSON.parse(localStorage.getItem("cart")) || [];
    const summaryData = JSON.parse(localStorage.getItem("checkoutSummary")) || null;
    
    if (cartData.length === 0 || !summaryData) {
        document.querySelector('.container').innerHTML = '<div class="error-message">السلة فارغة أو حدث خطأ في البيانات.</div>';
        return;
    }
    
    currentCart = cartData;
    finalTotalAmount = summaryData.finalTotal;
    shippingDetails = summaryData.shipping;
    
    // عرض قائمة المنتجات
    const itemsContainer = document.getElementById("orderItemsSummary");
    let productSubtotal = 0;
    itemsContainer.innerHTML = '<h4>المنتجات المطلوبة:</h4>';
    
    cartData.forEach(item => {
        const rawPrice = parseFloat(item.price.toString().replace(/[^0-9.]/g, '')) || 0;
        const itemTotal = rawPrice * item.qty;
        productSubtotal += itemTotal;
        
        itemsContainer.innerHTML += `
            <div class="cart-item">
                <span>${item.name}</span> (العدد: ${item.qty}) 
                - ${itemTotal.toLocaleString()} ر.ي
            </div>
        `;
    });
    
    // عرض الإجمالي
    document.getElementById("shippingSummary").textContent = shippingDetails.cost.toLocaleString() + " ر.ي (" + shippingDetails.location + ")";
    document.getElementById("finalTotalSummary").textContent = finalTotalAmount.toLocaleString() + " ر.ي";
  }

  // ------------------------------------------------------------------
  // 2. إرسال الطلب إلى Firebase
  // ------------------------------------------------------------------
  window.submitOrder = async function() {
    const name = document.getElementById("clientName").value.trim();
    const phone = document.getElementById("clientPhone").value.trim();
    const location = document.getElementById("clientLocation").value.trim();
    const btn = document.getElementById("submitOrderBtn");
    const errorBox = document.getElementById("errorMessage");

    if (!name || !phone || !location) {
        errorBox.textContent = "الرجاء تعبئة جميع حقول البيانات (الاسم، الجوال، العنوان).";
        errorBox.style.display = 'block';
        return;
    }
    
    errorBox.style.display = 'none';
    btn.textContent = "جاري تأكيد الطلب...";
    btn.disabled = true;

    try {
        await addDoc(ordersCol, {
            clientName: name,
            clientPhone: phone,
            clientLocation: location,
            orderTime: new Date().toISOString(),
            orderStatus: "جديد",
            
            // بيانات الطلب
            items: currentCart.map(item => ({
                id: item.id,
                name: item.name,
                qty: item.qty,
                price: item.price
            })),
            
            // ملخص الدفع
            shippingDetails: shippingDetails,
            finalTotal: finalTotalAmount
        });

        // مسح السلة من الذاكرة المحلية
        localStorage.removeItem("cart");
        localStorage.removeItem("checkoutSummary");
        
        // عرض رسالة النجاح
        document.getElementById('successMessage').style.display = 'block';
        document.querySelector('.info-form').style.display = 'none';
        document.querySelector('.summary-card').style.display = 'none';
        btn.style.display = 'none';

    } catch (e) {
        console.error("Error submitting order: ", e);
        errorBox.textContent = "❌ فشل إرسال الطلب! حاول مرة أخرى أو استخدم واتساب.";
        errorBox.style.display = 'block';
        btn.textContent = "تأكيد الطلب الآن";
        btn.disabled = false;
    }
  }

  loadCartSummary();
</script>

</body>
</html>
