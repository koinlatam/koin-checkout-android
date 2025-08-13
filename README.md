# Koin Checkout Android SDK
A lightweight Android library to integrate Koin's checkout experience into your app.

This SDK provides a streamlined and secure way to handle payments, supporting both modern Jetpack Compose UIs and traditional Android Views.

1. Installation
Add the Koin Checkout SDK to your project in two steps.

### Step 1: Add the Maven Repository
In your project's root settings.gradle.kts file, add the Koin public repository URL inside the dependencyResolutionManagement block:

```kotlin

// settings.gradle.kts

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        // Add Koin's repository here
        maven { url = uri("https://koinlatam.github.io/koin-checkout-android/") }
    }
}
```
### Step 2: Add the SDK Dependency
In your module's build.gradle.kts file (usually app/build.gradle.kts), add the implementation dependency:

```kotlin

// app/build.gradle.kts

dependencies {
    // ... other dependencies

    implementation("br.com.koin:payment-checkout:0.1.0") // Replace with the latest version
}
```

Sync your project with Gradle files to download the dependency.

2. Setup & Initialization
Before you can launch the checkout, you need to initialize the SDK with your credentials and configuration.

### Step 1: Add Internet Permission
Ensure your app has the INTERNET permission in your AndroidManifest.xml:

```XML

<manifest ...>
    <uses-permission android:name="android.permission.INTERNET" />

    <application ...>
        ...
    </application>
</manifest>
```
### Step 2: Initialize the SDK
Create a custom Application class to initialize the KoinPaymentCheckout. This only needs to be done once when your app starts.

The PaymentCheckoutConfig allows you to set your API key, customize colors, and define the environment (isHomolog).

```kotlin

// e.g., YourApplication.kt

import android.app.Application
import br.com.koin.payment.checkout.KoinPaymentCheckout
import br.com.koin.payment.checkout.core.PaymentCheckoutConfig

class YourApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        val config = PaymentCheckoutConfig.Builder(key = "YOUR_PRIVATE_API_KEY")
            .primaryColor("#008f21")      // Optional: Your brand's primary color
            .secondaryColor("#004E1D")    // Optional: Your brand's secondary color
            .currencyCode("BRL")          // Set the default currency
            .isHomolog(true)              // Use `true` for testing, `false` for production
            .build()

        KoinPaymentCheckout.initialize(config)
    }
}
```

### Step 3: Reference the Custom Application
Finally, reference your custom Application class in your AndroidManifest.xml:

```XML

<application
    android:name=".YourApplication"
    ...>
    </application>
```

3. Usage
Using the SDK involves two main parts: creating a payment request and then launching the checkout with that request.

### Step 1: Create a Payment Request
The SDK uses a type-safe Kotlin DSL (Domain-Specific Language) to build the PaymentRequest object. This ensures that all required fields are provided at compile time.

Here is an example of how to build a request:

```Kotlin

import br.com.koin.payment.checkout.features.launcher.data.model.paymentRequest

val request = paymentRequest {
    countryCode = "BR"
    ipAddress = "127.0.0.1"
    descriptor = "My Store"

    store {
        code = "your-store-code"
    }

    transaction {
        referenceId = "order-12345"
        account = "your-account-id"
        amount {
            currencyCode = "BRL"
            value = 150.99 // e.g., 150.99
        }
    }

    paymentMethod {
        code = "PIX" // e.g., PIX, CREDIT_CARD
    }

    payer {
        fullName = "João da Silva"
        email = "joao.silva@email.com"
        document {
            type = "CPF"
            number = "12345678900"
        }
    }

    items(
        {
            id = "product-01"
            name = "Product A"
            quantity = 1
            type = "PHYSICAL"
            category {
                id = "1"
                name = "Electronics"
            }
            price {
                currencyCode = "BRL"
                value = 150.99
            }
        }
    )
}
```

### Step 2: Launch the Checkout and Handle the Result
The SDK uses the standard Android ActivityResultLauncher to start the checkout flow and receive the result. The implementation differs slightly between Jetpack Compose and the traditional View system.

#### Using Jetpack Compose
In your Composable function, use rememberLauncherForActivityResult with the KoinPaymentCheckoutContract.

```kotlin

import androidx.activity.compose.rememberLauncherForActivityResult
import br.com.koin.payment.checkout.KoinPaymentCheckoutContract
import br.com.koin.payment.checkout.core.CheckoutResult

@Composable
fun MyPaymentScreen() {
    // Register the Activity Result Launcher
    val koinCheckoutLauncher = rememberLauncherForActivityResult(
        contract = KoinPaymentCheckoutContract()
    ) { result: CheckoutResult? ->
        // Handle the result here
        handleCheckoutResult(result)
    }

    Button(onClick = {
        // Create the request object as shown in the previous step
        val paymentRequest = createMyPaymentRequest() // Your function to build the request
        
        // Launch the SDK
        koinCheckoutLauncher.launch(paymentRequest)
    }) {
        Text("Pay Now")
    }
}

private fun handleCheckoutResult(result: CheckoutResult?) {
    when (result) {
        is CheckoutResult.Processed -> {
            // Success! The payment was processed.
            val status = result.data.status
            val referenceId = result.data.referenceId
            // Update your UI: show success message, navigate to next screen, etc.
        }
        is CheckoutResult.Failure -> {
            // An error occurred.
            val error = result.error
            // Update your UI: show an error dialog with details from `error`.
        }
        null -> {
            // The user cancelled the checkout flow (e.g., pressed the back button).
            // Update your UI accordingly.
        }
    }
}
```

#### Using Fragments or Activities (Views)
In your Fragment or Activity, declare and initialize the launcher, then call it from an event handler like a button click.

```kotlin

import androidx.activity.result.ActivityResultLauncher
import androidx.appcompat.app.AppCompatActivity
import br.com.koin.payment.checkout.KoinPaymentCheckoutContract
import br.com.koin.payment.checkout.core.CheckoutResult

class MyPaymentActivity : AppCompatActivity() {

    private lateinit var koinCheckoutLauncher: ActivityResultLauncher<PaymentRequest>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_payment)

        // Register the Activity Result Launcher
        koinCheckoutLauncher = registerForActivityResult(KoinPaymentCheckoutContract()) { result ->
            // Handle the result here
            handleCheckoutResult(result)
        }

        val payButton: Button = findViewById(R.id.pay_button)
        payButton.setOnClickListener {
            // Create the request object
            val paymentRequest = createMyPaymentRequest()

            // Launch the SDK
            koinCheckoutLauncher.launch(paymentRequest)
        }
    }

    private fun handleCheckoutResult(result: CheckoutResult?) {
        when (result) {
            is CheckoutResult.Processed -> {
                Log.d("KoinCheckout", "Processed: ${result.data.status}")
            }
            is CheckoutResult.Failure -> {
                Log.e("KoinCheckout", "Failure: ${result.error.message}")
            }
            null -> {
                Log.i("KoinCheckout", "Flow Canceled by user.")
            }
        }
    }
}
```

