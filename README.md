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

------
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

------
## Advanced retries

```swift

        let maxAttempts = 4
        
        let textSearch = searchInput.flatMap { text in
            return ApiController.shared.currentWeather(city: text)
                .do(onNext: { [weak self] data in
                    self?.cache[text] = data
                }).retryWhen { e in
                    return e.enumerated().flatMap { attempt, error -> Observable<Int> in
                        print("== retrying after \(attempt + 1) seconds ==")
                        if attempt >= maxAttempts - 1 {
                          return Observable.error(error)
                        }
                        return Observable<Int>.timer(.seconds(attempt + 1),
                                                     scheduler: MainScheduler.instance)
                                              .take(1)
                    }
                }
                .catchError { [weak self] error in
                    return Observable.just(self?.cache[text] ?? .empty)
                }
        }
```

```
== retrying after 1 seconds ==
... network ...
== retrying after 2 seconds ==
... network ...
== retrying after 3 seconds ==
... network ...

```

<img width="400" alt="スクリーンショット 2022-09-25 13 55 54" src="https://user-images.githubusercontent.com/47273077/192392208-280050bb-f189-47f8-b0bc-30065d85c8fa.png">

-----

## Creating custom errors
<img width="400" alt="スクリーンショット 2022-09-25 13 55 54" src="https://user-images.githubusercontent.com/47273077/192394795-54d85466-446d-4f9c-b218-a22811fc4d2e.png">

ApiController
```swift
  private func buildRequest(method: String = "GET", pathComponent: String, params: [(String, String)]) -> Observable<Data> {
    let request: Observable<URLRequest> = Observable.create { observer in
      let url = self.baseURL.appendingPathComponent(pathComponent)
      var request = URLRequest(url: url)
      let keyQueryItem = URLQueryItem(name: "appid", value: try? self.apiKey.value())
      let unitsQueryItem = URLQueryItem(name: "units", value: "metric")
      let urlComponents = NSURLComponents(url: url, resolvingAgainstBaseURL: true)!

      if method == "GET" {
        var queryItems = params.map { URLQueryItem(name: $0.0, value: $0.1) }
        queryItems.append(keyQueryItem)
        queryItems.append(unitsQueryItem)
        urlComponents.queryItems = queryItems
      } else {
        urlComponents.queryItems = [keyQueryItem, unitsQueryItem]

        let jsonData = try! JSONSerialization.data(withJSONObject: params, options: .prettyPrinted)
        request.httpBody = jsonData
      }

      request.url = urlComponents.url!
      request.httpMethod = method

      request.setValue("application/json", forHTTPHeaderField: "Content-Type")

      observer.onNext(request)
      observer.onCompleted()

      return Disposables.create()
    }

    let session = URLSession.shared
    return request.flatMap { request in
        return session.rx.response(request: request)
          .map { response, data in
          switch response.statusCode {
          case 200 ..< 300:
            return data
          case 400 ..< 500:
            throw ApiError.cityNotFound
          default:
            throw ApiError.serverFailure
          }
        }

    }
  }
```

ViewController
```swift
        let textSearch = searchInput.flatMap { text in
            return ApiController.shared.currentWeather(city: text)
                .do(onNext: { [weak self] data in
                    self?.cache[text] = data
                },
                onError: { error in
                    DispatchQueue.main.async { [weak self] in
                        guard let self = self else { return }
                        InfoView.showIn(viewController: self, message: "An error occurred")
                    }
                })
                .catchError { [weak self] error in
                    return Observable.just(self?.cache[text] ?? .empty)
                }
```
