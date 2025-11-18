# LaCrosse Weather App - Implementation Guide

## Project Setup

### Prerequisites
- Xcode 15.0+
- iOS 16.0+ deployment target
- Swift 5.9+
- LaCrosse View weather station with WiFi Gateway
- API credentials for your weather station

### Initial Project Structure
```
LaCrosseWeather/
├── LaCrosseWeather.xcodeproj
├── LaCrosseWeather/
│   ├── App/
│   │   ├── LaCrosseWeatherApp.swift
│   │   └── AppDelegate.swift
│   ├── Models/
│   │   ├── WeatherReading.swift
│   │   ├── WeatherForecast.swift
│   │   ├── SensorData.swift
│   │   └── TrendIndicators.swift
│   ├── ViewModels/
│   │   ├── DashboardViewModel.swift
│   │   ├── ForecastViewModel.swift
│   │   ├── SettingsViewModel.swift
│   │   └── HistoryViewModel.swift
│   ├── Views/
│   │   ├── Dashboard/
│   │   │   ├── DashboardView.swift
│   │   │   ├── CurrentConditionsCard.swift
│   │   │   ├── MetricsGrid.swift
│   │   │   └── HourlyForecastRow.swift
│   │   ├── Forecast/
│   │   │   ├── ForecastDetailView.swift
│   │   │   ├── TemperatureChart.swift
│   │   │   ├── PressureChart.swift
│   │   │   └── HourDetailCard.swift
│   │   ├── Settings/
│   │   │   ├── SettingsView.swift
│   │   │   ├── APIConfigurationView.swift
│   │   │   └── PreferencesView.swift
│   │   ├── History/
│   │   │   ├── HistoryView.swift
│   │   │   └── HistoryChart.swift
│   │   └── Components/
│   │       ├── WeatherIcon.swift
│   │       ├── TrendArrow.swift
│   │       ├── ConfidenceIndicator.swift
│   │       └── LoadingView.swift
│   ├── Services/
│   │   ├── WeatherAPIService.swift
│   │   ├── PredictionService.swift
│   │   ├── DataManager.swift
│   │   └── NotificationService.swift
│   ├── Utilities/
│   │   ├── Extensions/
│   │   │   ├── Date+Extensions.swift
│   │   │   ├── Double+Extensions.swift
│   │   │   └── Color+Extensions.swift
│   │   ├── Constants.swift
│   │   ├── NetworkError.swift
│   │   └── KeychainHelper.swift
│   ├── Resources/
│   │   ├── Assets.xcassets
│   │   └── Info.plist
│   └── CoreData/
│       ├── LaCrosseWeather.xcdatamodeld
│       └── PersistenceController.swift
└── LaCrosseWeatherTests/
    ├── ModelTests/
    ├── ViewModelTests/
    ├── ServiceTests/
    └── PredictionTests/
```

## Implementation Phases

### Phase 1: Core Setup (Week 1)

#### 1.1 Create Xcode Project
```bash
# Create new iOS App project
# Name: LaCrosseWeather
# Interface: SwiftUI
# Language: Swift
# Include Core Data: Yes
# Include Tests: Yes
```

#### 1.2 Configure Info.plist
```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <false/>
</dict>
<key>UIBackgroundModes</key>
<array>
    <string>fetch</string>
    <string>processing</string>
</array>
```

#### 1.3 Create Base Models
Start with [`WeatherReading.swift`](Models/WeatherReading.swift):
```swift
import Foundation

struct WeatherReading: Codable, Identifiable {
    let id: UUID
    let timestamp: Date
    let temperature: Double
    let humidity: Double
    let pressure: Double
    let windSpeed: Double
    let windDirection: Int
    let rainfall: Double
    let uvIndex: Double?
    
    init(id: UUID = UUID(), timestamp: Date = Date(), 
         temperature: Double, humidity: Double, pressure: Double,
         windSpeed: Double, windDirection: Int, rainfall: Double,
         uvIndex: Double? = nil) {
        self.id = id
        self.timestamp = timestamp
        self.temperature = temperature
        self.humidity = humidity
        self.pressure = pressure
        self.windSpeed = windSpeed
        self.windDirection = windDirection
        self.rainfall = rainfall
        self.uvIndex = uvIndex
    }
}
```

### Phase 2: API Integration (Week 1-2)

#### 2.1 Create API Service
Implement [`WeatherAPIService.swift`](Services/WeatherAPIService.swift):
```swift
import Foundation

class WeatherAPIService: ObservableObject {
    private let baseURL: String
    private let apiKey: String
    private let session: URLSession
    
    init(baseURL: String, apiKey: String) {
        self.baseURL = baseURL
        self.apiKey = apiKey
        
        let config = URLSessionConfiguration.default
        config.timeoutIntervalForRequest = 30
        config.timeoutIntervalForResource = 60
        self.session = URLSession(configuration: config)
    }
    
    func fetchCurrentWeather() async throws -> WeatherReading {
        let endpoint = "\(baseURL)/api/v1/current"
        guard let url = URL(string: endpoint) else {
            throw NetworkError.invalidURL
        }
        
        var request = URLRequest(url: url)
        request.setValue(apiKey, forHTTPHeaderField: "X-API-Key")
        request.httpMethod = "GET"
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        
        guard httpResponse.statusCode == 200 else {
            throw NetworkError.httpError(statusCode: httpResponse.statusCode)
        }
        
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return try decoder.decode(WeatherReading.self, from: data)
    }
    
    func fetchHistoricalData(hours: Int) async throws -> [WeatherReading] {
        let endpoint = "\(baseURL)/api/v1/history?hours=\(hours)"
        guard let url = URL(string: endpoint) else {
            throw NetworkError.invalidURL
        }
        
        var request = URLRequest(url: url)
        request.setValue(apiKey, forHTTPHeaderField: "X-API-Key")
        request.httpMethod = "GET"
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw NetworkError.invalidResponse
        }
        
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        return try decoder.decode([WeatherReading].self, from: data)
    }
}

enum NetworkError: LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(statusCode: Int)
    case decodingError
    case noData
    
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "Invalid API URL"
        case .invalidResponse:
            return "Invalid server response"
        case .httpError(let code):
            return "HTTP error: \(code)"
        case .decodingError:
            return "Failed to decode response"
        case .noData:
            return "No data received"
        }
    }
}
```

#### 2.2 Add Keychain Support
Create [`KeychainHelper.swift`](Utilities/KeychainHelper.swift) for secure API key storage.

### Phase 3: Prediction Engine (Week 2-3)

#### 3.1 Implement Trend Analysis
Create [`PredictionService.swift`](Services/PredictionService.swift):
```swift
import Foundation

class PredictionService {
    func generateForecast(from readings: [WeatherReading]) -> WeatherForecast {
        guard readings.count >= 12 else {
            return WeatherForecast.empty()
        }
        
        let trends = calculateTrends(from: readings)
        let predictions = generateHourlyPredictions(
            from: readings.last!,
            trends: trends,
            hours: 6
        )
        let confidence = calculateConfidence(from: readings, trends: trends)
        
        return WeatherForecast(
            id: UUID(),
            generatedAt: Date(),
            predictions: predictions,
            confidence: confidence,
            trendIndicators: trends
        )
    }
    
    private func calculateTrends(from readings: [WeatherReading]) -> TrendIndicators {
        let temperatureTrend = calculateTrend(
            values: readings.map { $0.temperature }
        )
        let pressureTrend = calculateTrend(
            values: readings.map { $0.pressure }
        )
        let humidityTrend = calculateTrend(
            values: readings.map { $0.humidity }
        )
        let windTrend = calculateTrend(
            values: readings.map { $0.windSpeed }
        )
        
        return TrendIndicators(
            temperatureTrend: temperatureTrend,
            pressureTrend: pressureTrend,
            humidityTrend: humidityTrend,
            windTrend: windTrend
        )
    }
    
    private func calculateTrend(values: [Double]) -> Trend {
        guard values.count >= 2 else { return .stable }
        
        let recentValues = Array(values.suffix(12))
        let slope = linearRegression(values: recentValues)
        
        if slope > 0.5 {
            return .rising
        } else if slope < -0.5 {
            return .falling
        } else {
            return .stable
        }
    }
    
    private func linearRegression(values: [Double]) -> Double {
        let n = Double(values.count)
        let x = Array(0..<values.count).map { Double($0) }
        let y = values
        
        let sumX = x.reduce(0, +)
        let sumY = y.reduce(0, +)
        let sumXY = zip(x, y).map(*).reduce(0, +)
        let sumX2 = x.map { $0 * $0 }.reduce(0, +)
        
        let slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX)
        return slope
    }
    
    private func generateHourlyPredictions(
        from current: WeatherReading,
        trends: TrendIndicators,
        hours: Int
    ) -> [HourlyPrediction] {
        var predictions: [HourlyPrediction] = []
        
        for hour in 1...hours {
            let tempChange = calculateTemperatureChange(
                hour: hour,
                trend: trends.temperatureTrend,
                currentTemp: current.temperature
            )
            
            let precipProb = calculatePrecipitationProbability(
                pressureTrend: trends.pressureTrend,
                humidity: current.humidity,
                hour: hour
            )
            
            let prediction = HourlyPrediction(
                hour: hour,
                temperature: TemperatureRange(
                    low: current.temperature + tempChange - 1,
                    high: current.temperature + tempChange + 1
                ),
                humidity: HumidityRange(
                    low: max(0, current.humidity - 5),
                    high: min(100, current.humidity + 5)
                ),
                precipitationProbability: precipProb,
                conditions: determineConditions(
                    precipProb: precipProb,
                    humidity: current.humidity
                )
            )
            
            predictions.append(prediction)
        }
        
        return predictions
    }
    
    private func calculateTemperatureChange(
        hour: Int,
        trend: Trend,
        currentTemp: Double
    ) -> Double {
        let baseChange: Double
        switch trend {
        case .rising:
            baseChange = 0.5 * Double(hour)
        case .falling:
            baseChange = -0.5 * Double(hour)
        case .stable:
            baseChange = 0
        }
        
        return baseChange * 0.8
    }
    
    private func calculatePrecipitationProbability(
        pressureTrend: Trend,
        humidity: Double,
        hour: Int
    ) -> Double {
        var probability = 0.0
        
        if pressureTrend == .falling {
            probability += 30.0
        }
        
        if humidity > 80 {
            probability += 20.0
        } else if humidity > 70 {
            probability += 10.0
        }
        
        probability = min(probability, 90.0)
        return probability
    }
    
    private func determineConditions(
        precipProb: Double,
        humidity: Double
    ) -> WeatherCondition {
        if precipProb > 60 {
            return .rainy
        } else if precipProb > 30 || humidity > 75 {
            return .cloudy
        } else if humidity < 60 {
            return .sunny
        } else {
            return .partlyCloudy
        }
    }
    
    private func calculateConfidence(
        from readings: [WeatherReading],
        trends: TrendIndicators
    ) -> Double {
        var confidence = 100.0
        
        let variance = calculateVariance(readings: readings)
        confidence -= variance * 10
        
        let trendStability = assessTrendStability(trends: trends)
        confidence *= trendStability
        
        return max(min(confidence, 100.0), 0.0)
    }
    
    private func calculateVariance(readings: [WeatherReading]) -> Double {
        let temps = readings.map { $0.temperature }
        let mean = temps.reduce(0, +) / Double(temps.count)
        let squaredDiffs = temps.map { pow($0 - mean, 2) }
        return sqrt(squaredDiffs.reduce(0, +) / Double(temps.count))
    }
    
    private func assessTrendStability(trends: TrendIndicators) -> Double {
        let stableCount = [
            trends.temperatureTrend,
            trends.pressureTrend,
            trends.humidityTrend
        ].filter { $0 == .stable }.count
        
        return 0.7 + (Double(stableCount) * 0.1)
    }
}
```

### Phase 4: UI Implementation (Week 3-4)

#### 4.1 Create Dashboard View
Implement [`DashboardView.swift`](Views/Dashboard/DashboardView.swift) with:
- Current conditions card
- Metrics grid
- Hourly forecast scroll
- Pull-to-refresh

#### 4.2 Create ViewModels
Implement [`DashboardViewModel.swift`](ViewModels/DashboardViewModel.swift):
```swift
import Foundation
import Combine

@MainActor
class DashboardViewModel: ObservableObject {
    @Published var currentWeather: WeatherReading?
    @Published var forecast: WeatherForecast?
    @Published var isLoading = false
    @Published var error: Error?
    
    private let apiService: WeatherAPIService
    private let predictionService: PredictionService
    private var cancellables = Set<AnyCancellable>()
    private var refreshTimer: Timer?
    
    init(apiService: WeatherAPIService, predictionService: PredictionService) {
        self.apiService = apiService
        self.predictionService = predictionService
        setupAutoRefresh()
    }
    
    func loadWeatherData() async {
        isLoading = true
        error = nil
        
        do {
            let current = try await apiService.fetchCurrentWeather()
            let historical = try await apiService.fetchHistoricalData(hours: 6)
            
            currentWeather = current
            forecast = predictionService.generateForecast(from: historical)
        } catch {
            self.error = error
        }
        
        isLoading = false
    }
    
    private func setupAutoRefresh() {
        refreshTimer = Timer.scheduledTimer(
            withTimeInterval: 300,
            repeats: true
        ) { [weak self] _ in
            Task {
                await self?.loadWeatherData()
            }
        }
    }
}
```

### Phase 5: Data Persistence (Week 4)

#### 5.1 Configure Core Data
Set up [`LaCrosseWeather.xcdatamodeld`](CoreData/LaCrosseWeather.xcdatamodeld) with entities:
- WeatherReadingEntity
- PredictionEntity

#### 5.2 Implement Data Manager
Create [`DataManager.swift`](Services/DataManager.swift) for:
- Saving weather readings
- Caching predictions
- Historical data queries
- Data cleanup

### Phase 6: Settings & Configuration (Week 5)

#### 6.1 Create Settings View
Implement [`SettingsView.swift`](Views/Settings/SettingsView.swift) with:
- API configuration
- Unit preferences
- Refresh interval
- Data management

#### 6.2 Add UserDefaults Support
Store user preferences:
- Temperature unit (F/C)
- Wind speed unit (mph/kph)
- Pressure unit (hPa/inHg)
- Refresh interval

### Phase 7: Charts & Visualization (Week 5-6)

#### 7.1 Implement Charts
Use Swift Charts framework for:
- Temperature trend charts
- Pressure trend charts
- Humidity trend charts
- Historical data visualization

#### 7.2 Create Custom Components
Build reusable components:
- Weather icons
- Trend arrows
- Confidence indicators
- Loading states

### Phase 8: Testing (Week 6)

#### 8.1 Unit Tests
Test:
- Prediction algorithm accuracy
- Trend calculation
- API response parsing
- Data model validation

#### 8.2 Integration Tests
Test:
- API service communication
- Core Data operations
- ViewModel state management

#### 8.3 UI Tests
Test:
- Navigation flow
- Settings configuration
- Error handling
- Offline mode

### Phase 9: Polish & Optimization (Week 7)

#### 9.1 Performance Optimization
- Implement caching strategy
- Optimize chart rendering
- Reduce memory footprint
- Background fetch optimization

#### 9.2 Accessibility
- VoiceOver support
- Dynamic Type
- Color contrast
- Haptic feedback

#### 9.3 Error Handling
- Graceful degradation
- Retry logic
- User-friendly messages
- Logging

### Phase 10: App Store Preparation (Week 8)

#### 10.1 App Store Assets
- App icon (1024x1024)
- Screenshots (all device sizes)
- App preview video
- Description and keywords

#### 10.2 Privacy & Compliance
- Privacy policy
- Terms of service
- Data collection disclosure
- Location permissions (if needed)

## Key Implementation Notes

### API Integration Best Practices
1. Always use HTTPS
2. Store API keys securely in Keychain
3. Implement request timeout handling
4. Add retry logic with exponential backoff
5. Cache responses appropriately

### Prediction Algorithm Tips
1. Require minimum 2 hours of data
2. Use weighted averages for recent data
3. Account for time of day effects
4. Validate predictions against actual data
5. Adjust confidence based on data quality

### Performance Considerations
1. Use async/await for network calls
2. Implement proper cancellation
3. Limit Core Data fetch sizes
4. Use lazy loading for historical data
5. Optimize chart rendering

### Testing Strategy
1. Mock API responses for unit tests
2. Test edge cases (no data, errors)
3. Verify prediction accuracy
4. Test offline functionality
5. Performance testing with large datasets

## Deployment Checklist

- [ ] All features implemented and tested
- [ ] No compiler warnings
- [ ] App Store assets prepared
- [ ] Privacy policy created
- [ ] TestFlight beta testing completed
- [ ] Performance profiling done
- [ ] Accessibility audit passed
- [ ] App Store submission ready

## Maintenance Plan

### Regular Updates
- Monitor API changes
- Update prediction algorithm
- Fix reported bugs
- Add user-requested features

### Analytics
- Track prediction accuracy
- Monitor API response times
- Measure user engagement
- Identify crash patterns

### Future Enhancements
- Widget support
- Watch app
- Notifications
- Multiple stations
- Machine learning predictions