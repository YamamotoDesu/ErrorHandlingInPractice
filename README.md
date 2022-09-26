# ErrorHandlingInPractice
Catching errors

```swift
    private var cache = [String: Weather]()

    override func viewDidLoad() {
        let textSearch = searchInput.flatMap { text in
          return ApiController.shared.currentWeather(city: text)
            .do(onNext: { [weak self] data in
              self?.cache[text] = data
            })
            .catchError { error in
                return Observable.just(self.cache[text] ?? .empty)
            }
        }

```
![image](https://user-images.githubusercontent.com/47273077/192276278-0821865d-32d2-4d11-a0b4-9b4b2eaf8352.png)
