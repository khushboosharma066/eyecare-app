# eyecare-app
import SwiftUI
import ARKit
import Vision
import CoreHaptics

struct ContentView: View {
    var body: some View {
        TabView {
            FocusTrackingView()
                .tabItem {
                    Label("Focus", systemImage: "eye.fill")
                }
                
            EyeExercisesView()
                .tabItem {
                    Label("Exercises", systemImage: "list.bullet")
                }
                
            HomeRemediesView()
                .tabItem {
                    Label("Remedies", systemImage: "cross.fill")
                }
                
            CameraAnalysisView()
                .tabItem {
                    Label("Scan", systemImage: "camera.fill")
                }
                
            DashboardView()
                .tabItem {
                    Label("Dashboard", systemImage: "chart.bar.fill")
                }
        }
        .accentColor(.blue)
    }
}
import UIKit
import CoreVideo

extension UIImage {
    func pixelBuffer() -> CVPixelBuffer? {
        let size = CGSize(width: 224, height: 224) // Resize the image to match the model's expected input size
        UIGraphicsBeginImageContextWithOptions(size, false, 0)
        self.draw(in: CGRect(origin: .zero, size: size))
        guard let cgImage = UIGraphicsGetImageFromCurrentImageContext()?.cgImage else { return nil }
        UIGraphicsEndImageContext()
        
        var pixelBuffer: CVPixelBuffer?
        let options: [CFString: Any] = [
            kCVPixelBufferCGImageCompatibilityKey: true,
            kCVPixelBufferCGBitmapContextCompatibilityKey: true
        ]
        
        let status = CVPixelBufferCreate(
            kCFAllocatorDefault,
            Int(size.width),
            Int(size.height),
            kCVPixelFormatType_32BGRA,
            options as CFDictionary,
            &pixelBuffer
        )
        
        guard status == kCVReturnSuccess, let buffer = pixelBuffer else {
            return nil
        }
        
        let context = CGContext(
            data: CVPixelBufferGetBaseAddress(buffer),
            width: Int(size.width),
            height: Int(size.height),
            bitsPerComponent: 8,
            bytesPerRow: CVPixelBufferGetBytesPerRow(buffer),
            space: CGColorSpaceCreateDeviceRGB(),
            bitmapInfo: CGImageAlphaInfo.noneSkipFirst.rawValue
        )
        
        context?.draw(cgImage, in: CGRect(origin: .zero, size: size))
        return buffer
    }
}
import CoreML
import Vision

class FocusTrackingViewModel: ObservableObject {
    @Published var isFocused = true
    @Published var focusScore: Int = 100
    @Published var audioPlayer: AVAudioPlayer?
    
    private var model: Resnet50?
    
    init() {
        // Load the model (ensure resnet50.mlpackage is added correctly)
        if let modelURL = Bundle.main.url(forResource: "resnet50", withExtension: "mlpackage") {
            do {
                model = try Resnet50(contentsOf: modelURL)
            } catch {
                print("Error loading model: \(error.localizedDescription)")
            }
        }
    }
    
    func playDistractionSound() {
        guard let soundURL = Bundle.main.url(forResource: "distraction_alarm", withExtension: "mp3") else { return }
        
        do {
            audioPlayer = try AVAudioPlayer(contentsOf: soundURL)
            audioPlayer?.play()
        } catch {
            print("Error playing sound: \(error.localizedDescription)")
        }
    }
    
    func predictFocusUsingModel(image: UIImage) {
        guard let model = model, let pixelBuffer = image.pixelBuffer() else { return }
        
        do {
            // Perform prediction using Core ML model
            let prediction = try model.prediction(image: pixelBuffer)
            // Assuming the model output is a dictionary with predicted class
            print("Prediction: \(prediction.classLabel)")
            
            // Here you can change focusScore based on the prediction result
            // For simplicity, assuming "focused" = high score, "distracted" = low score
            if prediction.classLabel == "focused" {
                focusScore = 90
                isFocused = true
            } else {
                focusScore = 50
                isFocused = false
            }
        } catch {
            print("Error making prediction: \(error.localizedDescription)")
        }
    }
}
import SwiftUI
import AVFoundation

struct FocusTrackingView: View {
    @StateObject private var viewModel = FocusTrackingViewModel() // Observed ViewModel

    @State private var image: UIImage? = UIImage(named: "sampleImage") // Placeholder image, replace with real-time camera image
    
    var body: some View {
        ZStack {
            // Colorful Gradient Background
            LinearGradient(gradient: Gradient(colors: [Color.gray.opacity(0.3), Color.purple.opacity(0.1), Color.gray.opacity(0.1)]), startPoint: .top, endPoint: .bottom)
                .edgesIgnoringSafeArea(.all)
            
            VStack {
                Text("Focus Tracking")
                    .font(.largeTitle)
                    .fontWeight(.bold)
                    .foregroundColor(.white)
                    .padding()
                
                Spacer()
                
                // Round Focus Tracker
                ZStack {
                    Circle()
                        .fill(viewModel.isFocused ? Color.gray.opacity(0.5) : Color.gray)
                        .frame(width: 250, height: 250)
                        .shadow(radius: 10)
                        .scaleEffect(viewModel.isFocused ? 1.0 : 1.1)  // Scaling effect when distracted
                        .animation(.easeInOut(duration: 0.5))
                    
                    VStack {
                        Text(viewModel.isFocused ? "You are Focused" : "You are Distracted")
                            .font(.title2)
                            .foregroundColor(.white)
                            .padding()
                            .animation(.easeInOut(duration: 0.5)) // Animation for text change
                        
                        Text("Focus Score: \(viewModel.focusScore)%")
                            .font(.title3)
                            .padding()
                            .background(RoundedRectangle(cornerRadius: 15).fill(Color.white).shadow(radius: 5))
                            .padding(.top, 10)
                            .foregroundColor(.black)
                    }
                }
                
                Spacer()
            }
            .onAppear {
                // Start tracking and predicting on a timer (you can use camera feed here)
                Timer.scheduledTimer(withTimeInterval: 2.0, repeats: true) { _ in
                    // Simulate focus/distracted states
                    let newFocus = Bool.random()
                    viewModel.isFocused = newFocus
                    viewModel.focusScore = newFocus ? Int.random(in: 85...100) : Int.random(in: 40...70)
                    
                    // Play sound if distracted
                    if !viewModel.isFocused {
                        viewModel.playDistractionSound()
                    }
                    
                    // Make prediction using Core ML model
                    if let image = self.image {
                        viewModel.predictFocusUsingModel(image: image)
                    }
                }
            }
        }
    }
}


struct EyeExercise: Identifiable {
    let id = UUID()
    let disease: String
    let exercise: String
}

struct EyeExercisesView: View {
    let eyeDiseases: [EyeExercise] = [
        EyeExercise(disease: "Dry Eyes", exercise: "Blinking Exercise: Blink rapidly for 20 seconds."),
        EyeExercise(disease: "Eye Strain", exercise: "20-20-20 Rule: Look 20 feet away for 20 seconds every 20 minutes."),
        EyeExercise(disease: "Myopia", exercise: "Focus Shifting: Shift focus between near and far objects."),
        EyeExercise(disease: "Hyperopia", exercise: "Pencil Push-ups: Hold a pencil and bring it close while focusing."),
        EyeExercise(disease: "Astigmatism", exercise: "Eye Rolling: Roll eyes in circles to relax muscles."),
        EyeExercise(disease: "Glaucoma", exercise: "Palming: Rub hands together and place over closed eyes for warmth."),
        EyeExercise(disease: "Computer Vision Syndrome", exercise: "Candle Exercise: Focus on a candle flame for a few minutes to improve focus."),
        EyeExercise(disease: "Presbyopia", exercise: "Hand Massage: Gently massage around the eyes to relax muscles."),
        EyeExercise(disease: "Cataracts", exercise: "Sun Gazing: Look at the early morning sun for a few seconds to improve eye strength."),
        EyeExercise(disease: "Night Blindness", exercise: "Carrot & Vitamin A Intake: Improve night vision by consuming foods rich in vitamin A.")
    ]
    
    @State private var selectedExercise: EyeExercise?
    @State private var searchText = ""
    
    var filteredDiseases: [EyeExercise] {
        if searchText.isEmpty {
            return eyeDiseases
        } else {
            return eyeDiseases.filter { $0.disease.localizedCaseInsensitiveContains(searchText) }
        }
    }
    
    var body: some View {
        NavigationView {
            VStack {
                TextField("Search disease...", text: $searchText)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding()
                
                ScrollView {
                    LazyVStack(alignment: .leading, spacing: 15) {
                        ForEach(filteredDiseases) { disease in
                            Button(action: {
                                selectedExercise = disease
                            }) {
                                Text(disease.disease)
                                    .font(.headline)
                                    .foregroundColor(.black)
                                    .padding()
                                    .frame(maxWidth: .infinity, alignment: .leading)
                                    .background(RoundedRectangle(cornerRadius: 10).fill(Color.white).shadow(radius: 2))
                            }
                            .padding(.horizontal)
                        }
                    }
                    .padding(.top, 10)
                }
                .background(Color.gray.opacity(0.2))
                .frame(maxHeight: .infinity)
            }
            .navigationTitle("Eye Exercises")
            .background(Color.gray.opacity(0.2).edgesIgnoringSafeArea(.all))
            .fullScreenCover(item: $selectedExercise) { exercise in
                ExerciseDetailView(exercise: exercise.exercise)
            }
        }
    }
}

struct ExerciseDetailView: View {
    let exercise: String
    
    var body: some View {
        VStack(spacing: 20) {
            Text("Exercise Details")
                .font(.largeTitle)
                .padding()
            
            Text(exercise)
                .font(.title2)
                .padding()
                .background(RoundedRectangle(cornerRadius: 10).fill(Color.white).shadow(radius: 2))
            
            Spacer()
            
            Button("Close") {
                UIApplication.shared.windows.first?.rootViewController?.dismiss(animated: true, completion: nil)
            }
            .padding()
            .background(RoundedRectangle(cornerRadius: 10).fill(Color.blue))
            .foregroundColor(.white)
        }
        .padding()
        .background(Color.gray.opacity(0.2).edgesIgnoringSafeArea(.all))
    }
}
import SwiftUI

struct HomeRemediesView: View {
    // Sample data for eye diseases and remedies
    let remedies = [
        Remedy(disease: "Dry Eyes", remedy: "Use a warm compress or artificial tears to relieve dryness."),
        Remedy(disease: "Eye Strain", remedy: "Follow the 20-20-20 rule: Take a break every 20 minutes, look at something 20 feet away for 20 seconds."),
        Remedy(disease: "Myopia", remedy: "Try eye exercises like focusing on a distant object and then back to a nearby one."),
        Remedy(disease: "Astigmatism", remedy: "Wear corrective lenses or glasses with the appropriate prescription."),
        Remedy(disease: "Glaucoma", remedy: "Regular eye checkups, proper medication, and eye drops to manage intraocular pressure."),
        Remedy(disease: "Computer Vision Syndrome", remedy: "Adjust your screen brightness, use anti-glare screens, and blink more often."),
        Remedy(disease: "Cataracts", remedy: "Consult with an ophthalmologist for surgical options."),
        Remedy(disease: "Night Blindness", remedy: "Increase Vitamin A intake by consuming more carrots and leafy greens.")
    ]
    
    @State private var selectedRemedy: Remedy?
    @State private var searchText = ""
    
    var filteredRemedies: [Remedy] {
        if searchText.isEmpty {
            return remedies
        } else {
            return remedies.filter { $0.disease.localizedCaseInsensitiveContains(searchText) }
        }
    }
    
    var body: some View {
        NavigationView {
            VStack {
                TextField("Search disease...", text: $searchText)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding()
                
                ScrollView {
                    LazyVStack(alignment: .leading, spacing: 15) {
                        ForEach(filteredRemedies) { remedy in
                            Button(action: {
                                selectedRemedy = remedy
                            }) {
                                Text(remedy.disease)
                                    .font(.headline)
                                    .foregroundColor(.black)
                                    .padding()
                                    .frame(maxWidth: .infinity, alignment: .leading)
                                    .background(RoundedRectangle(cornerRadius: 10).fill(Color.white).shadow(radius: 2))
                            }
                            .padding(.horizontal)
                        }
                    }
                    .padding(.top, 10)
                }
                .background(Color.gray.opacity(0.2))
                .frame(maxHeight: .infinity)
            }
            .navigationTitle("Home Remedies")
            .background(Color.gray.opacity(0.2).edgesIgnoringSafeArea(.all))
            .fullScreenCover(item: $selectedRemedy) { remedy in
                RemedyDetailView(remedy: remedy.remedy)
            }
        }
    }
}

struct Remedy: Identifiable {
    let id = UUID()
    let disease: String
    let remedy: String
}

struct RemedyDetailView: View {
    let remedy: String
    
    var body: some View {
        VStack(spacing: 20) {
            Text("Remedy Details")
                .font(.largeTitle)
                .padding()
            
            Text(remedy)
                .font(.title2)
                .padding()
                .background(RoundedRectangle(cornerRadius: 10).fill(Color.white).shadow(radius: 2))
            
            Spacer()
            
            Button("Close") {
                UIApplication.shared.windows.first?.rootViewController?.dismiss(animated: true, completion: nil)
            }
            .padding()
            .background(RoundedRectangle(cornerRadius: 10).fill(Color.blue))
            .foregroundColor(.white)
        }
        .padding()
        .background(Color.gray.opacity(0.2).edgesIgnoringSafeArea(.all))
    }
}
import SwiftUI
import AVFoundation
import Vision
import CoreML

// MARK: - Main View
struct CameraAnalysisView: View {
    @StateObject private var eyeTracker = EyeTracker()
    
    var body: some View {
        MainView(eyeTracker: eyeTracker)
    }
}

// MARK: - Main View
struct MainView: View {
    @ObservedObject var eyeTracker: EyeTracker
    @State private var showingBreakAlert = false
    
    var body: some View {
        ZStack {
            Color.gray.edgesIgnoringSafeArea(.all)
            
            VStack(spacing: 20) {
                // Camera View or Setup View
                if eyeTracker.isSetup {
                    CameraView(eyeTracker: eyeTracker)
                } else {
                    SetupView(eyeTracker: eyeTracker)
                }
                
                // Status View
                VStack(spacing: 15) {
                    StatusView(screenTime: eyeTracker.screenTime)
                    EyeStatusView(status: eyeTracker.eyeStatus)
                }
                .padding()
                .background(Color.white.opacity(0.8))
                .cornerRadius(12)
                
                // Controls
                HStack(spacing: 20) {
                    Button("Reset Timer") {
                        eyeTracker.resetScreenTime()
                    }
                    .buttonStyle(PrimaryButtonStyle())
                    
                    Button(eyeTracker.session.isRunning ? "Stop Camera" : "Start Camera") {
                        eyeTracker.toggleCamera()
                    }
                    .buttonStyle(PrimaryButtonStyle())
                }
            }
            .padding()
        }
        .alert("Time for a Break!", isPresented: $showingBreakAlert) {
            Button("OK", role: .cancel) { }
        } message: {
            Text("Look at something 20 feet away for 20 seconds.")
        }
        .onReceive(eyeTracker.$shouldTakeBreak) { shouldBreak in
            showingBreakAlert = shouldBreak
        }
    }
}

// MARK: - Supporting Views
struct CameraView: View {
    @ObservedObject var eyeTracker: EyeTracker
    
    var body: some View {
        ZStack {
            Color.gray.opacity(0.1)
            
            if eyeTracker.session.isRunning {
                CameraPreview(session: eyeTracker.session)
            } else {
                Text("Camera Stopped")
                    .foregroundColor(.white)
            }
        }
        .frame(height: 300)
        .cornerRadius(12)
    }
}

struct SetupView: View {
    @ObservedObject var eyeTracker: EyeTracker
    
    var body: some View {
        VStack(spacing: 20) {
            Image(systemName: "camera.fill")
                .font(.system(size: 50))
                .foregroundColor(.green.opacity(0.1))
            
            Text("Camera Access Required")
                .font(.headline)
                .foregroundColor(.white)
            
            Button("Setup Camera") {
                eyeTracker.setupCamera()
            }
            .buttonStyle(PrimaryButtonStyle())
        }
        .frame(height: 300)
        .frame(maxWidth: .infinity)
        .background(Color.gray.opacity(0.3))
        .cornerRadius(12)
    }
}

struct StatusView: View {
    let screenTime: TimeInterval
    
    var body: some View {
        HStack {
            Image(systemName: "clock")
            Text("Screen Time: \(formatTime(screenTime))")
        }
        .foregroundColor(.white)
        .font(.headline)
    }
    
    private func formatTime(_ seconds: TimeInterval) -> String {
        let hours = Int(seconds) / 3600
        let minutes = Int(seconds) / 60 % 60
        return String(format: "%02d:%02d", hours, minutes)
    }
}

struct EyeStatusView: View {
    let status: String
    
    var body: some View {
        HStack {
            Image(systemName: status == "Comfortable" ? "eye" : "eye.slash")
            Text("Status: \(status)")
        }
        .foregroundColor(status == "Comfortable" ? .green : .orange)
        .font(.headline)
    }
}

// MARK: - Button Style
struct PrimaryButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .foregroundColor(.white)
            .padding()
            .background(Color.blue.opacity(0.9))
            .cornerRadius(8)
            .opacity(configuration.isPressed ? 0.7 : 1)
    }
}

// MARK: - Camera Preview
struct CameraPreview: UIViewRepresentable {
    let session: AVCaptureSession
    
    func makeUIView(context: Context) -> UIView {
        let view = UIView(frame: .zero)
        let previewLayer = AVCaptureVideoPreviewLayer(session: session)
        previewLayer.videoGravity = .resizeAspectFill
        view.layer.addSublayer(previewLayer)
        return view
    }
    
    func updateUIView(_ uiView: UIView, context: Context) {
        if let previewLayer = uiView.layer.sublayers?.first as? AVCaptureVideoPreviewLayer {
            previewLayer.frame = uiView.bounds
        } else {
            let previewLayer = AVCaptureVideoPreviewLayer(session: session)
            previewLayer.frame = uiView.bounds
            previewLayer.videoGravity = .resizeAspectFill
            uiView.layer.addSublayer(previewLayer)
        }
    }
}

// MARK: - Eye Tracker
class EyeTracker: NSObject, ObservableObject, AVCaptureVideoDataOutputSampleBufferDelegate {
    @Published var isSetup = false
    @Published var screenTime: TimeInterval = 0
    @Published var eyeStatus = "Checking..."
    @Published var shouldTakeBreak = false
    
    let session = AVCaptureSession()
    private var timer: Timer?
    
    override init() {
        super.init()
        
        #if !targetEnvironment(simulator)
        checkCameraAuthorization()
        #else
        DispatchQueue.main.async {
            self.isSetup = true // Simulate setup for preview mode
        }
        #endif
    }
    
    // Check camera authorization and set up
    func checkCameraAuthorization() {
        switch AVCaptureDevice.authorizationStatus(for: .video) {
        case .authorized:
            setupCamera()
        case .notDetermined:
            AVCaptureDevice.requestAccess(for: .video) { [weak self] granted in
                if granted {
                    DispatchQueue.main.async {
                        self?.setupCamera()
                    }
                }
            }
        default:
            break
        }
    }
    
    // Setup the camera
    func setupCamera() {
        #if targetEnvironment(simulator)
        DispatchQueue.main.async {
            self.isSetup = true // Simulate camera setup in simulator
        }
        return
        #endif
        
        guard let device = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .front) else {
            print("No camera found")
            return
        }
        
        do {
            let input = try AVCaptureDeviceInput(device: device)
            let output = AVCaptureVideoDataOutput()
            output.setSampleBufferDelegate(self, queue: DispatchQueue.global(qos: .userInteractive))
            
            session.beginConfiguration()
            if session.canAddInput(input) && session.canAddOutput(output) {
                session.addInput(input)
                session.addOutput(output)
                session.commitConfiguration()
                
                DispatchQueue.main.async { [weak self] in
                    self?.isSetup = true
                    self?.startTracking()
                }
            }
        } catch {
            print("Camera setup failed: \(error)")
        }
    }
    
    // Toggle camera start/stop
    func toggleCamera() {
        if session.isRunning {
            session.stopRunning()
        } else {
            session.startRunning()
        }
    }
    
    // Reset screen time
    func resetScreenTime() {
        screenTime = 0
    }
    
    // Start tracking
    func startTracking() {
        print("Start eye tracking")
    }
    
    // AVCaptureVideoDataOutputSampleBufferDelegate method
    func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
        // Implement frame processing logic here
    }
}



import SwiftUI
import ARKit
import CoreML
import Vision

class RealTimeFocusViewModel: NSObject, ObservableObject, ARSessionDelegate {
    @Published var focusScore: Int = 0
    @Published var eyeStrainLevel: Int = 0
    
    private var session: ARSession?
    private var focusModel: VNCoreMLModel?
    private var eyeStrainModel: VNCoreMLModel?
    
    override init() {
        super.init()
        setupARSession()
        loadModels()
    }
    
    func setupARSession() {
        // Setup ARKit session
        let configuration = ARFaceTrackingConfiguration()
        session = ARSession()
        session?.delegate = self
        session?.run(configuration)
    }
    
    func loadModels() {
        // Load your pre-trained CoreML models for focus and eye strain
        guard let focusModelURL = Bundle.main.url(forResource: "FocusModel", withExtension: "mlmodelc"),
              let eyeStrainModelURL = Bundle.main.url(forResource: "EyeStrainModel", withExtension: "mlmodelc") else {
            return
        }
        
        do {
            let focusModel = try VNCoreMLModel(for: MLModel(contentsOf: focusModelURL))
            let eyeStrainModel = try VNCoreMLModel(for: MLModel(contentsOf: eyeStrainModelURL))
            self.focusModel = focusModel
            self.eyeStrainModel = eyeStrainModel
        } catch {
            print("Error loading models: \(error)")
        }
    }
    
    func updateFocusAndEyeStrain() {
        // Example of passing camera data to CoreML model and updating values
        
        // Placeholder code: You would have to implement real-time AR data processing.
        let randomFocus = Int.random(in: 60...100)
        let randomStrain = Int.random(in: 1...5)
        
        // Update focus score and eye strain level
        self.focusScore = randomFocus
        self.eyeStrainLevel = randomStrain
    }
    
    // MARK: - ARSessionDelegate
    func session(_ session: ARSession, didUpdate frame: ARFrame) {
        // Process camera frame with CoreML models to get focus and eye strain levels
        updateFocusAndEyeStrain()
    }
}

struct RealTimeFocusView: View {
    @StateObject private var viewModel = RealTimeFocusViewModel()
    
    var body: some View {
        VStack {
            Text("Focus Score: \(viewModel.focusScore)%")
                .font(.title)
                .padding()
            
            Text("Eye Strain Level: \(viewModel.eyeStrainLevel)")
                .font(.title2)
                .padding()
            
            // Add more real-time stats or visual elements as needed
        }
        .onAppear {
            // Start the AR session and real-time processing when the view appears
            viewModel.setupARSession()
        }
    }
}

struct DashboardView: View {
    var body: some View {
        NavigationView {
            RealTimeFocusView()
                .navigationTitle("Focus & Eye Strain")
        }
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
