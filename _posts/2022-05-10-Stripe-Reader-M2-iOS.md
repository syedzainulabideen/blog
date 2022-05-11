---
 layout: post
 title:  "Stripe: Accept in-person payments using Reader M2 - iOS"
 date:   2022-05-11 23:45:00 +0500
 categories: stripe
---
 
Hey Readers, In this tutorial we are looking into the Integration of Stripe’s In-Person Payment using Stripe Reader M2. Below are the quick overview of steps, we will be following.

### Steps Overview:

---

- [ ]  **Create Locations:** To organize your readers, you need to create locations e.g. Every reader is linked to the specific location of your physical store location.
    
    You can either create location using [Stripe Dashboard](https://dashboard.stripe.com/terminal/locations) or using the Terminal’s SDK by following [here](https://stripe.com/docs/terminal/fleet/locations#create-a-location-with-standard-connect) 
    
- [ ]  **Create Connection Token:** First you need to create connection token for the Terminal SDK. Best approach for this step is to create connection token on backend and access created token through API on frontend.
    
    Detail instructions on this as below:
    
    1. Create new iOS app and install Stripe Terminal SDK using Cocopods. 
        
        ```swift
        pod 'StripeTerminal', '~> 2.0'
        ```
        
    2. Also add permissions to the .plist file, as per Apple requirements.
        
        ```swift
        <key>NSLocationWhenInUseUsageDescription</key><string>Location access is required in order to accept payments.</string>
        <key>UIBackgroundModes</key><array><string>bluetooth-central</string></array>
        <key>NSBluetoothPeripheralUsageDescription</key><string>Bluetooth access is required in order to connect to supported bluetooth card readers.</string>
        <key>NSBluetoothAlwaysUsageDescription</key><string>This app uses Bluetooth to connect to supported card readers.</string>
        ```
        
    3. Create a new swift file as APIClient.swift and add below contents to newly created file. Make sure you have selected the current Location, where Reader is being used. 
        
        ```swift
        import StripeTerminal
        
        class APIClient: NSObject, ConnectionTokenProvider {
            static let shared = APIClient()
            
            func fetchConnectionToken(_ completion: @escaping ConnectionTokenCompletionBlock) {
                guard let selectedLocationId = "{{CURRENT_SELECTED_LOCATION_ID}}" else { return } 
        				
        				let url = "https://example.com/v1/payments/terminal/token"
                guard let validURL = URL(string: url) else {
                    completion(nil, NSError.createError(2000, domain: "com.stripe-terminal-ios.example", description: "Invalid URL"))
                    return
                }
                let config = URLSessionConfiguration.default
                let session = URLSession(configuration: config)
                var request = URLRequest(url: validURL)
                
                // Id of the current selected Location, where you are using the terminal
                let body = [
                    "location": selectedLocationId
                ]
                request.httpMethod = "POST"
                request.addValue("{{AUTHORIZATION_KEY_VALUE}}", forHTTPHeaderField: "Authorization")
                request.addValue("application/json", forHTTPHeaderField: "Content-Type")
                request.httpBody = body.jsonData
                
                let task = session.dataTask(with: request) { (data, response, error) in
                    if let validData = data {
                        do {
                            let responseObject = try JSONDecoder().decode(CreateConnectTokenResponse.self, from: validData)
                            if let secret = responseObject.data?.secret {
                                completion(secret, nil)
                            }
                            else {
                                completion(nil, NSError.createError(2000, domain: "com.stripe-terminal-ios.example", description: "Missing 'secret' in ConnectionToken JSON response"))
                            }
                        } catch {
                            completion(nil, NSError.createError(2000, domain: "com.stripe-terminal-ios.example", description: "Unable to parse the response data."))
                        }
                    } else {
                        completion(nil, NSError.createError(1000, domain: "com.stripe-terminal-ios.example", description: "No data in response from ConnectionToken endpoint"))
                    }
                }
                
                task.resume()
            }
        }
        ```
        
    4. Now link above created class to the Stripe Terminal SDK in Appdelegate file, as below
        
        ```swift
        func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey : Any]? = nil) -> Bool {
        	// ----
           Terminal.setTokenProvider(APIClient.shared)
        	// ----
        }
        ```
        
- [ ]  **Discover and Connect with Reader:** In this step we will be creating a shared class to control complete flow of the discovering, connecting and making payment.
    1. Create new file TerminalManager.swift then add below content step wise.
        
        ```swift
        import StripeTerminal
        
        class TerminalManager: NSObject, ObservableObject {
            static let shared = TerminalManager()
        
        		var connectedReader:Reader?
        }
        ```
        
    2. Add new property to handle the current connection status of the reader and try to discover readers if not already connected.
        
        ```swift
        var currentConnectionStatus:ConnectionStatus = Terminal.shared.connectionStatus {
                didSet {
                    switch currentConnectionStatus {
                    case .notConnected:
                        self.connectedReader = nil
                        let config = DiscoveryConfiguration(discoveryMethod: .bluetoothScan, simulated: false)
                        self.discoverCancelable = Terminal.shared.discoverReaders(config, delegate: self) { error in
                            if let error = error {
                                print("**terminal-manager** discoverReaders failed: \(error.localizedDescription)")
                            }
                        }
                    case .connected:
                        print("Reader successfully connected.")
                        break
                    case .connecting:
                        self.connectedReader = nil
                        break
                    @unknown default:
                        break
                    }
                }
            }
        ```
        
    3. Add below function in TerminalManager class and You need to call this function before proceed with Reader connection or payment, so Terminal SDK can be setup for delegates. 
        
        ```swift
        func setUpInitialReaderFlow() {
            Terminal.shared.delegate = self
            self.currentConnectionStatus = Terminal.shared.connectionStatus
        }
        ```
        
        To handle the discover result, you need to add delegates functions of the Terminal. Below delegate function didUpdateDiscoveredReaders is being used to get the discovered readers. If Reader is already setup with some location Id, you can use that otherwise you need to provide your current selected location’s id.
        
        ```swift
        extension TerminalManager: DiscoveryDelegate  {
            
            func terminal(_ terminal: Terminal, didUpdateDiscoveredReaders readers: [Reader]) {
                guard let selectedReader = readers.first else { return }
                        guard terminal.connectionStatus == .notConnected else { return }
                
                let currentLocation = selectedReader.locationId ?? {{CURRENT_SELECTED_LOCATION_ID}}
                
                let connectionConfig = BluetoothConnectionConfiguration(
                    locationId: currentLocation!
                )
                Terminal.shared.connectBluetoothReader(selectedReader, delegate: self, connectionConfig: connectionConfig) { reader, error in
                    if let _ = reader {
                        self.connectedReader = reader
                    } else if let error = error {
        							print("\(error.localizedDescription)")
                    }
                }
            }
        }
        ```
        
        Once you have successfully connected to the Reader, you can then proceed to collect payment from User. Details in next steps.
        

- [ ]  **Creating Payment Intent:**  As a best practice, we need to create payment on the server side using the amount, currency etc. And then fetch created Intent object using API. Add below API call in APIClient.swift class to fetch server created Payment Intent Response object.
    
    ```swift
    typealias CreateIntentCompletion = ((ServiceIntentResponse.IntentDataClass?, String?) -> Void)
    
    func createPaymentIntent(_ amount:Int, completion: @escaping CreateIntentCompletion) {
            let url = "https://example/com/v1/payments/terminal/create-intent"
            guard let validURL = URL(string: url) else {
                completion(nil, "Invalid URL")
                return
            }
            let config = URLSessionConfiguration.default
            let session = URLSession(configuration: config)
            var request = URLRequest(url: validURL)
    
            let body = [
                "amount": amount,
                "currency": "usd"
            ] as [String : Any]
            
            request.httpMethod = "POST"
            request.addValue("{{AUTHORIZATION_KEY_VALUE}}", forHTTPHeaderField: "Authorization")
            request.addValue("application/json", forHTTPHeaderField: "Content-Type")
            request.httpBody = body.jsonData
    
            let task = session.dataTask(with: request) { (data, response, error) in
                if let validData = data {
                    do {
                        let responseObject = try JSONDecoder().decode(ServiceIntentResponse.self, from: validData)
                        if let object = responseObject.data, responseObject.status == 200 {
    												// need to save this created Payment Intent to use in next steps.
                            AppViewModel.shared.createIntentResponse = object
                            completion(object, nil)
                        } else {
                            completion(nil, responseObject.message ?? "Unable to process request, error code: \(responseObject.status ?? 0)")
                        }
                    }
                    catch(let exception) {
                        completion(nil, exception.localizedDescription)
                    }
                } else {
                    completion(nil, "No data in response from ConnectionToken endpoint")
                }
            }
    
            task.resume()
        }
    ```
    

- [ ]  **Collect Payment Method:** Once you successfully created and reterive the payment Intetn, now it’s time to ask User to Tap/Swipe/Insert the Card.
    
    ```swift
    guard let selectedRate = {{AMOUNT_OF_SELECTED_RATE}} else { return }
    
    APIClient.shared.createPaymentIntent(selectedRate) { responseObject, errorString in
           guard let validObject = responseObject else {
                print("Error while creating Payment Intent")
                return
            }
            guard let jsonData = try? DictionaryEncoder.encode(validObject) else {
                print("Unable to convert JSON to Payment Intent")
                return
            }
            
            guard let paymentIntent = PaymentIntent.decodedObject(fromJSON:jsonData) else {
                print("Unable to create Payment intent from response object")
                return
            }
            
            Terminal.shared.collectPaymentMethod(paymentIntent) { collectResult, collectError in
                if let error = collectError {
                    print("Unable to create payment: \(error.localizedDescription)")
                } else if let validPaymentIntent = collectResult {
    								// Next Step
                    self.processPayment(validPaymentIntent)
                }
            }
     }
    ```
    
- [ ]  **Processing Payment:** After successfully collecting the Payment method, once User Tap/Swipe card on the Reader. You can now process the response Payment Intent, as below.
    
    ```swift
    Terminal.shared.processPayment(paymentIntent) { processResult, processError in
        if let error = processError {
            print"\(error.localizedDescription)")
        } else if let processPaymentPaymentIntent = processResult {
            print("*** processPayment succeeded")
        }
    }
    ```
    
- [ ]  **Capture Payment:** Once you process the Payment Intent, final step would be to Capture the actual payment using that Payment Intent’s Id. And that will be done on Server using API call.
    
    Add below function to the APIClient.swift file.
    
    ```swift
    func capturePaymentIntent(_ paymentIntentId:String, completion: @escaping CapturePaymentCompletion) {
            let url = "https://example.com/v1/payments/terminal/capture-intent"
            guard let validURL = URL(string: url) else {
                completion(nil, "Invalid URL")
                return
            }
            let config = URLSessionConfiguration.default
            let session = URLSession(configuration: config)
            var request = URLRequest(url: validURL)
    
            let body = [
                "payment_intent": paymentIntentId
            ]
            request.httpMethod = "POST"
            request.addValue("{{AUTHORIZATION_KEY_VALUE}}", forHTTPHeaderField: "Authorization")
            request.addValue("application/json", forHTTPHeaderField: "Content-Type")
            request.httpBody = body.jsonData
    
            let task = session.dataTask(with: request) { (data, response, error) in
                if let validData = data {
                    do {
                        let responseObject = try JSONDecoder().decode(ServiceIntentResponse.self, from: validData)
                        if let object = responseObject.data, responseObject.status == 200 {
                            completion(object, nil)
                        } else {
                            completion(nil, responseObject.message ?? "Unable to process request, error code: \(responseObject.status ?? 0)")
                        }
                    }
                    catch(let exception) {
                        completion(nil, exception.localizedDescription)
                    }
                } else {
                    completion(nil, "No data in response from ConnectionToken endpoint")
                }
            }
    
            task.resume()
        }
    ```
    
    That’s it, You have completed the basic step for the integration of Stripe Reader M2 to your iOS app. Cheers 🙂
    But..
    
    There are still some important points that we need to consider while implementing the above flow. I’ll explain them below.
    

### Important Considerations:

---

- [ ]  **Reader Connectivity:**
    
    Stripe’s Reader M2 goes to the sleep mode in which you wouldn’t be able to discover it using the SDK functions. So in this case you have to manually press the side button on the Reader to awake it from sleep mode.
    
- [ ]  **Cancellable Intents:**
Sometimes, Terminal SDK failed the process in any step, either Create Payment Intent or Collection Payment Method Intent. So if you want to retry that specific step again, you can reuse that Intent (for example user use different card if first card declined). But If you want to create new Intent again, so in this case you need to cancel already created Intents. (for example if user go back and change the selected Rate.) 
Below is the example of using cancellable intent for creation (if collectPayment already generated and user go back to select new rate). We first check if intent exist, cancel and then create payment intent again for new rate.
    
    ```swift
    var collectPaymentCancelable:PaymentIntent?
    
    if let paymentCancelable = self.collectPaymentCancelable {
        self.readerCurrentState = .cancelExistingIntent
        paymentCancelable.cancel { error in
            if let validError = error {
                self.readerCurrentState = .error(message: validError.localizedDescription, showRetry: true)
            }
            else {
                self.createNewPaymentIntent()
            }
        }
    }
    else {
        self.createNewPaymentIntent()
    }
    ```
    
- [ ]  **SDK is busy with other command:**
    
    More often you will get the error from SDK somthing like this ‘**Could not execute discoverReaders because the SDK is busy with another command: discoverReaders.** ’
    
    At this point you need to step back from code and reevaluate your whole flow. You shouldn’t call **discoverReaders** command if Terminal SDK is already executing this command. Second thing, in some other cases as we discussed above that If Terminal SDK is already waiting for User to swipe card after creating the Collect Payment Intent and if User go back at this point. We must cancel the Already created Intents. So when user come back to make payment, he shouldn’t get this error.
    

### Final Words:

---

So, Congratulation for making to the end of this tutorial. Hopefully you have got some headsup to move foward with Stripe Reader’s Integration with you iOS App. There must be some corner cases that still remains undiscussed in this tutorail. Let me know if you want me to cover any specific topic related to the Stripe Readers.
