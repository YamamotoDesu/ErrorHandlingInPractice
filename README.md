# ErrorHandlingInPractice
## Catching errors

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
<img width="400" alt="スクリーンショット 2022-09-25 13 55 54" src="https://user-images.githubusercontent.com/47273077/192276278-0821865d-32d2-4d11-a0b4-9b4b2eaf8352.png">

## Retrying on error

```swift
        
        let textSearch = searchInput.flatMap { text in
          return ApiController.shared.currentWeather(city: text)
            .do(onNext: { [weak self] data in
              self?.cache[text] = data
            })
            .retry(3)
            .catchError { [weak self] error in
              return Observable.just(self?.cache[text] ?? .empty)
            }
        }
```

<img width="400" alt="スクリーンショット 2022-09-25 13 55 54" src="https://user-images.githubusercontent.com/47273077/192278742-f6928f13-dbc7-48c4-aa1c-191126dd729f.png">


