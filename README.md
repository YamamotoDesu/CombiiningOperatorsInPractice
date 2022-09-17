# CombiiningOperatorsInPractice

## RxSwift: Reactive Programming with Swift | raywenderlich.com
![image](https://user-images.githubusercontent.com/47273077/185172130-b3557025-c636-4a1b-8490-c900c8312b77.png)

![image](https://user-images.githubusercontent.com/47273077/190841193-ef11315a-9e62-4855-88e6-cbdb510dcb3b.png)

## 1. Generic request technique(Model)
```swift
  static func request<T: Decodable>(endpoint: String, query: [String: Any] = [:], contentIdentifier: String) -> Observable<T> {
    do {
      guard let url = URL(string: API)?.appendingPathComponent(endpoint),
            var components = URLComponents(url: url, resolvingAgainstBaseURL: true) else {
        throw EOError.invalidURL(endpoint)
      }
      
      components.queryItems = try query.compactMap { (key, value) in
        guard let v = value as? CustomStringConvertible else {
          throw EOError.invalidParameter(key, value)
        }
        return URLQueryItem(name: key, value: v.description)
      }
      
      guard let finalURL = components.url else {
        throw EOError.invalidURL(endpoint)
      }
      
      let request = URLRequest(url: finalURL)

      return URLSession.shared.rx.response(request: request)
        .map { (result: (response: HTTPURLResponse, data: Data)) -> T in
          let decoder = self.jsonDecoder(contentIdentifier: contentIdentifier)
          let envelope = try decoder.decode(EOEnvelope<T>.self, from: result.data)
          return envelope.content
        }
      
    } catch {
      return Observable.empty()
    }
    
  }
```
  
## 2. Fetch categories(Model)
```swift
  static var categories: Observable<[EOCategory]> = {
    let request: Observable<[EOCategory]> = EONET.request(endpoint: categoriesEndpoint, contentIdentifier: "categories")

      return request
        .map { categories in categories.sorted { $0.name < $1.name } }
        .catchErrorJustReturn([])
        .share(replay: 1, scope: .forever)
    }()
 ```
    
##  3 . Adding the event download service(Model)
```swift
  private static func events(forLast days: Int, closed: Bool) -> Observable<[EOEvent]> {
    let query: [String: Any] = [
      "days": days,
      "status": (closed ? "closed" : "open")
    ]
    let request: Observable<[EOEvent]> = EONET.request(endpoint: eventsEndpoint, query: query, contentIdentifier: "events")
    return request.catchErrorJustReturn([])
  }
```

![image](https://user-images.githubusercontent.com/47273077/190841321-8900a07a-f379-44ef-8960-0693ac46ba5e.png)

## 4 . Downloading in parallel
```swift
  static func events(forLast days: Int = 360) -> Observable<[EOEvent]> {
    let openEvents = events(forLast: days, closed: false)
    let closedEvents = events(forLast: days, closed: true)

//    return openEvents.concat(closedEvents)
    
    // Downloading in parallel
    return Observable.of(openEvents, closedEvents)
      .merge()
      .reduce([]) { running, new in
        running + new
      }

  }
 ```
 ![image](https://user-images.githubusercontent.com/47273077/190841614-3412f2e4-a2c3-40af-b4bc-18df310158ce.png)


------

## ViewController

CategoriesViewController
```swift
import UIKit
import RxSwift
import RxCocoa

class CategoriesViewController: UIViewController, UITableViewDataSource, UITableViewDelegate {

  @IBOutlet var tableView: UITableView!
  
  let categories = BehaviorRelay<[EOCategory]>(value: [])
  let disposeBag = DisposeBag()

  override func viewDidLoad() {
    super.viewDidLoad()
    
    categories
      .asObservable()
      .subscribe(onNext: { [weak self] _ in
        DispatchQueue.main.async {
          self?.tableView?.reloadData()
        }
      })
      .disposed(by: disposeBag)


    startDownload()
  }

  func startDownload() {
//    let eoCategories = EONET.categories
//    eoCategories
//      .bind(to: categories)
//      .disposed(by: disposeBag)
    
    let eoCategories = EONET.categories
    let downloadedEvents = EONET.events(forLast: 360)
    
    let updatedCategories = Observable
      .combineLatest(eoCategories, downloadedEvents) {
        (categories, events) -> [EOCategory] in
        
        return categories.map { category in
          var cat = category
          cat.events = events.filter {
            $0.categories.contains(where: { $0.id == category.id })
          }
          return cat
        }
      }
    
    eoCategories
      .concat(updatedCategories)
      .bind(to: categories)
      .disposed(by: disposeBag)

  }
  
  // MARK: UITableViewDataSource
  func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return categories.value.count
  }
  
  func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "categoryCell")!
    
    let category = categories.value[indexPath.row]
    cell.textLabel?.text = "\(category.name) (\(category.events.count))"
    cell.accessoryType = (category.events.count > 0) ? .disclosureIndicator : .none
    cell.detailTextLabel?.text = category.description

    return cell
  }
  
}

```

![image](https://user-images.githubusercontent.com/47273077/190841372-a2ee2506-fef3-4380-b5be-7d20715a71fc.png)

## 2 . Event View Controller

EventsViewController
```swift
import UIKit
import RxSwift
import RxCocoa

class EventsViewController: UIViewController, UITableViewDataSource {

  @IBOutlet var tableView: UITableView!
  @IBOutlet var slider: UISlider!
  @IBOutlet var daysLabel: UILabel!
  
  let events = BehaviorRelay<[EOEvent]>(value: [])
  let disposeBag = DisposeBag()

  override func viewDidLoad() {
    super.viewDidLoad()

    tableView.rowHeight = UITableView.automaticDimension
    tableView.estimatedRowHeight = 60
    
    events.asObservable()
      .subscribe(onNext: { [weak self] _ in
        self?.tableView.reloadData()
      })
      .disposed(by: disposeBag)

  }

  @IBAction func sliderAction(slider: UISlider) {
  }

  // MARK: UITableViewDataSource
  func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return events.value.count
  }

  func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "eventCell") as! EventCell
    let event = events.value[indexPath.row]
    cell.configure(event: event)

    return cell
  }

}
```

CategoriesViewController
```swift 
func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    let category = categories.value[indexPath.row]
    tableView.deselectRow(at: indexPath, animated: true)

    guard !category.events.isEmpty else { return }

    let eventsController = storyboard!.instantiateViewController(withIdentifier: "events") as! EventsViewController
    eventsController.title = category.name
    eventsController.events.accept(category.events)
    navigationController!.pushViewController(eventsController, animated: true)
  }
```


----

## Wiring the days selector

<img width="300" src="https://user-images.githubusercontent.com/47273077/190842307-7c4c70a9-3d97-4ce4-8cc1-b6b50a56d9e4.gif">

```swift
import UIKit
import RxSwift
import RxCocoa

class EventsViewController: UIViewController, UITableViewDataSource {

  @IBOutlet var tableView: UITableView!
  @IBOutlet var slider: UISlider!
  @IBOutlet var daysLabel: UILabel!
  
  let events = BehaviorRelay<[EOEvent]>(value: [])
  let disposeBag = DisposeBag()
  
  let days = BehaviorRelay<Int>(value: 360)
  let filteredEvents = BehaviorRelay<[EOEvent]>(value: [])

  override func viewDidLoad() {
    super.viewDidLoad()

    tableView.rowHeight = UITableView.automaticDimension
    tableView.estimatedRowHeight = 60
    
    events.asObservable()
      .subscribe(onNext: { [weak self] _ in
        self?.tableView.reloadData()
      })
      .disposed(by: disposeBag)
    
    Observable.combineLatest(days, events) { days, events -> [EOEvent] in
      let maxInterval = TimeInterval(days * 24 * 3600)
        return events.filter { event in
          if let date = event.date {
            return abs(date.timeIntervalSinceNow) < maxInterval
          }
        return true
      }
    }
    .bind(to: filteredEvents)
    .disposed(by: disposeBag)
    
    filteredEvents.asObservable()
      .subscribe(onNext: { _ in
        DispatchQueue.main.async { [weak self] in
          self?.tableView.reloadData()
        }
      })
      .disposed(by: disposeBag)

    
    days.asObservable()
      .subscribe(onNext: { [weak self] days in
        self?.daysLabel.text = "Last \(days) days"
      })
      .disposed(by: disposeBag)


  }

  @IBAction func sliderAction(slider: UISlider) {
    days.accept(Int(slider.value))
  }

  // MARK: UITableViewDataSource
  func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return filteredEvents.value.count
  }

  func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "eventCell") as! EventCell
    let event = filteredEvents.value[indexPath.row]
    cell.configure(event: event)

    return cell
  }

}

```

