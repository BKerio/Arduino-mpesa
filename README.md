# PoolPay - ESP32 M-Pesa Payment Terminal

PoolPay is an IoT-based payment terminal system. It allows users to enter a phone number and payment amount using a physical keypad, displays prompts on an I2C LCD, and triggers a Safaricom M-Pesa STK Push via a custom Node.js Express backend. 

Instead of relying on fragile WebSocket connections, the ESP32 performs highly reliable HTTP polling against the backend database status endpoint to track transaction outcomes (Success, Cancelled, Wrong PIN, Insufficient Funds, etc.) in real-time.

---

##  Hardware Connections & Wiring

### 1. 4x3 Matrix Keypad
| Keypad Pin | ESP32 GPIO | Description |
| :--- | :--- | :--- |
| **R1** | **14** | Row 1 |
| **R2** | **27** | Row 2 |
| **R3** | **26** | Row 3 |
| **R4** | **25** | Row 4 |
| **C1** | **33** | Column 1 |
| **C2** | **32** | Column 2 |
| **C3** | **13** | Column 3 |

### 2. 16x2 I2C LCD Display
| LCD Pin | ESP32 Pin | Description |
| :--- | :--- | :--- |
| **VCC** | **5V** | Power Supply |
| **GND** | **GND** | Ground |
| **SDA** | **21** | I2C Data |
| **SCL** | **22** | I2C Clock |

---

##  Project Structure

```text
├── arduino/
│   └── mpesa_stkpush.cpp  # ESP32 Arduino Core source code
├── backend/
│   ├── src/
│   │   ├── index.ts       # Express server entry point & Socket.IO server
│   │   ├── routes/
│   │   │   └── mpesa.ts   # Safaricom Daraja STK Push & Callback HTTP endpoints
│   │   └── models/
│   │       └── Payments.ts# MongoDB Transaction Schema
│   ├── .env.example       # Example template for Daraja credentials
│   └── tsconfig.json      # TypeScript compiler config
└── README.md              # Project Documentation
```

---

##  Getting Started

### 1. Backend Setup
The backend is built with Node.js, Express, TypeScript, and MongoDB.

1. Navigate to the `backend/` directory:
   ```bash
   cd backend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Create a `.env` file from the example:
   ```bash
   cp .env.example .env
   ```
4. Fill in your Daraja credentials, MongoDB connection string, and local server port:
   ```env
   PORT=5000
   MONGO_URI=mongodb://localhost:27017/mpesa
   MPESA_CONSUMER_KEY=your_consumer_key
   MPESA_CONSUMER_SECRET=your_consumer_secret
   MPESA_PASSKEY=your_stk_push_passkey
   MPESA_SHORTCODE=174379
   TILL_NO=174379
   MPESA_TRANSACTIONTYPE=CustomerPayBillOnline
   MPESA_CALLBACK_URL=https://your-public-domain.ngrok-free.app/api/stkpush/callback
   MPESA_BASE_URL=https://sandbox.safaricom.co.ke
   ```
   >  **Note:** Safaricom requires a public HTTPS callback URL. Use a tool like **ngrok** to tunnel your local port `5000` to the internet (`ngrok http 5000`) and set `MPESA_CALLBACK_URL` accordingly.
5. Start the development server:
   ```bash
   npm run dev
   ```

### 2. ESP32 Firmware Configuration
1. Open [arduino/mpesa_stkpush.cpp](file:///c:/Users/brian/Mpesa/arduino/mpesa_stkpush.cpp) in the Arduino IDE (rename extension to `.ino` if using the classic IDE) or build using PlatformIO.
2. Ensure you have the following libraries installed:
   * **Keypad** by Mark Stanley, Alexander Brevig
   * **LiquidCrystal_I2C** by Frank de Brabander
   * **ArduinoJson** by Benoit Blanchon (v6.x)
3. Set your Wi-Fi credentials:
   ```cpp
   const char* ssid = "Your SSID";
   const char* password = "Your Password";
   ```
4. Change the backend API URLs to match your server's local network IP address (e.g., `192.168.100.148`):
   ```cpp
   const char* api_stkpush = "http://192.168.100.148:5000/api/stkpush";
   const char* api_status_base = "http://192.168.100.148:5000/api/stkpush/status/";
   ```
5. Compile and upload to your ESP32 board!

---

## User Workflow
1. **Idle State**: LCD displays `Enter Phone`.
2. **Phone Number Entry**: Input the customer's phone number (e.g. `0717*****480`) and press `#` to proceed.
   * If you make a mistake, press `*` to clear and start over.
3. **Amount Entry**: LCD displays `Amount:`. Key in the payment amount and press `#`.
4. **Push Sent**: ESP32 hits `/api/stkpush`, triggers Safaricom, receives the checkout request ID, and changes the LCD to `Check Phone & Pay`.
5. **State Lock**: Keypad input is disabled during payment processing.
6. **Polling**: ESP32 queries the backend status endpoint every 3 seconds.
7. **Result Dispatch**: Once the user enters their PIN on their phone, the callback updates MongoDB, and the ESP32 registers the payment status on its next poll:
   * **SUCCESS**
   * **CANCELLED**
   * **WRONG PIN**
   * **INSUFFICIENT FUNDS**
8. After displaying the outcome for 3 seconds, the terminal resets back to `Enter Phone`.
