+++
title = "[UIKit]UIKit(CollectionView)+Combine"
date = 2023-02-25T15:16:36+09:00
draft = false
tags = ["alamofire", "combine", "ios", "ios 13.0+", "network", "swift", "ui", "uikit"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/65"
+++

# CollectionView에 Combine 적용하기

`RxSwift` + `RxCocoa` 조합처럼 보통 `Combine` + `SwiftUI` 조합으로 많이 쓰인다고 한다.

그런데 회사에서는 `UIKit`을 사용중인데 `Combine` + `UIKit` 조합은 어떻게 사용하나 공부하다가 자주 사용하는 `CollectionView`에 적용해 데이터 변경에 자동으로 반응하는 UI를 테스트해 보았다.

테스트는 다음과 같은 단계로 진행했다.

1. `UICollectionViewDiffableDataSource`를 따르는 `UICollectionView`를 만든다

1. API를 요청하여 데이터를 방출하는 `Combine Publisher`를 만든다.

1. `Publisher`를 구독하고, 데이터가 수신되면 `CollectionView`의 DataSource를 업데이트한다.

1. `CollectionView`에 업데이트 된 데이터를 표시한다.

## DiffableDataSource를 사용한 이유

- Publisher가 방출하는 데이터의 변화에 유연하게 대응하기 용이하기 때문에 `DiffableDataSource`를 사용

- 참고 : [개발자 문서](https://developer.apple.com/documentation/uikit/uicollectionviewdiffabledatasource)

사실 `Combine`이나 `Diffable`이나 다 공부하려고 사용했다. 공부를 위해 UI도 코드로 그렸다.

# UICollectionView

## CollectionView 기본 레이아웃 설정

- `DiffableDataSource`를 통해 `CollectionView`를 구현하는 방법은 [Modern Collection View](/posts/2023-02-21-modern-collection-view/) 을 참고.]

- 해당 프로젝트에서는 `UICollectionViewDiffableDataSource`의 section과 item의 타입을 설정하여 상속해 사용했다.

```swift
class MoviesCollectionViewDiffableDataSource: UICollectionViewDiffableDataSource<String?, Movies> { }
```

## Combine Publisher

### Model

- API 로 요청한 데이터 값을 `CombinePublisher`로 만드는 단계

![](/images/tistory/65/01.webp)

- API는 [imdb-api](https://imdb-api.com)라는 곳의 영화 검색 API를 사용했다. 회원가입 후 받는 API Key를 받고 아래와 같은 형태의 URL 로 호출하면 된다. URL: https://imdb-api.com/en/API/SearchMovie/<API Key>/<검색할 영화 제목 >

- 해당 API를 호출해 받게 되는 결과값의 포맷에 맞게 Model을 작성

```swift
struct Result: Codable {
  let searchType: String
  let expression: String
  let results: [Movies]
  let errorMessage: String
}

struct Movies: Codable, Hashable {
  let id: String
  let resultType: String
  let image: String
  let title, description: String
}
```

- Model을 작성할 때에 Cell에 필요한 데이터의 정보는 Hashable 프로토콜을 준수해야한다. UICollectionViewDiffableDataSource의 ItemIdentifierType이 셀을 구분할 때에 Hash값을 가지고 구분하나보다.

![](/images/tistory/65/02.webp)

*UICollectionViewDiffableDataSource의 제네릭으로 들어가는 타입*

### API request

- `Alamofire`를 사용했다. `@escaping`이나 `async`/`await`를 사용해도 되지만 기왕 `Combine`을 사용하기로 한거 `Combine`으로 처리했다.

```swift
var cancellable = Set<AnyCancellable>() // 메모리 해제를 위한 cansellable

func fetchMovies(of keyword: String) -> AnyPublisher<[Movies], Error> {
  // 1. url 주소 생성. 실패시 Fail로 publisher 반환
	guard let url = URL(string: BASE_URL + API_KEY + "/" + keyword) else {
    return Fail(error: URLError(.badURL))
           .eraseToAnyPublisher()
  }

	// 2. completion Handler를 @escaping closure 대신 Future로 사용
  return Future() { promise in
    AF.request(url)
      .publishDecodable(type: Result.self)      // 값을 디코딩함
      .value()                                  // 디코딩된 값을 AnyPublisher<Data, AFError> 형태로
      .sink { completion in
        switch completion{
        case .finished:
          print("fetchMovies finished")
        case .failure(let error):
          print("fetchMovies error: \(error)")
          promise(.failure(URLError(.badServerResponse)))    // 실패할 경우 에러값 publisher 반환
        }
      } receiveValue: { result in
        promise(.success(result.results))        // 성공한 경우 데이터값을 담은 publisher 반환
      }
      .store(in: &self.cancellable)              // 작업 후 메모리 해제
  }
  .eraseToAnyPublisher()                         // 최종적으로 AnyPubliser로 반환
}
```

# Publisher를 구독하고 diffableDataSource에 업데이트

### ViewModel

```swift
class ViewModel {
  var cancellables = Set<AnyCancellable>() // 메모리 해제를 위한 Cancellable
  @Published var keyword: String = ""       // 1.

  var diffableDataSource: MoviesCollectionViewDiffableDataSource!
  var snapshot = NSDiffableDataSourceSnapshot<String?, Movies>()

  init() {
    $keyword.receive(on: RunLoop.main)
    .debounce(for: .seconds(0.5), scheduler: RunLoop.main)  // 2.
    .sink { (_) in
      self.searchMovies()
    }.store(in: &cancellables)
  }

  func searchMovies() {
		// APIService의 FetchMovies를 구독
    APIService.shared.fetchMovies(of: keyword)
      .sink { completion in
        switch completion {
        case .finished:
          print("ViewModel searchMovies finished")
        case .failure(let error):
          print("ViewModel searchMovies failure: \(error.localizedDescription)")
        }
      } receiveValue: { movies in
				// api를 통해 값을 받으면 해당 데이터를 diffableDataSource에 넣는다.
        self.snapshot.deleteAllItems()
        self.snapshot.appendSections([""])

        if movies.isEmpty {
					// 3.
          self.diffableDataSource.apply(self.snapshot, animatingDifferences: true)
          return
        }
        self.snapshot.appendItems(movies)
        self.diffableDataSource.apply(self.snapshot, animatingDifferences: true)
      }
      .store(in: &cancellables)
  }
}
```

1. `@Published` : `Combine`에서 제공하는 Property wrapper로 값의 변화를 감지하고 `didset`에서 이벤트를 방출한다.

1. `debounce` : duetime으로부터 해당 시간만큼 데이터를 받지 않으면 최근 데이터를 방출한다.

# CollectionView에 데이터를 적용

```swift
@Published var keyStroke: String = ""
var viewModel = ViewModel()

func setupObservers() {
		// 1.
    $keyStroke
      .receive(on: RunLoop.main)
      .sink { keyword in
        print(keyword)
        self.viewModel.keyword = keyword
      }
      .store(in: &cancellables)

		// 2.
    viewModel.diffableDataSource = MoviesCollectionViewDiffableDataSource(collectionView: collectionView) { (collectionView, indexPath, itemIdentifier) -> UICollectionViewCell? in
      guard let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "MainCell", for: indexPath) as? MainCell else { return UICollectionViewCell() }

      cell.movie = itemIdentifier
      return cell
    }
  }
```

1. `@Published` 로 선언된 keyStroke의 값이 바뀔 때마다 **ViewModel**로 해당 값을 할당한다. keyStroke는 `UISearchBarDelegate`를 통해 입력된 값을 할당 받는다. 이를 구독하여**ViewModel**의 keyword에 준다.

- **ViewModel**로 방출된 데이터는**ViewModel**의 keyword 또한` @Published`로 선언되어 있으므로 값이 변할때 `debounce` 시간을 설정하여 `sink`한다. `sink`가 성공적으로 이루어지면 API를 호출하며, API의 결과값을 `DataSource`에 저장한다.

1. APIService로 받은 Movies의 데이터를 ViewModel**로 넘겨줌**

```swift
var movie: Movies! {
  didSet {
    setupData()
  }
}

private func setupData() {
  name.text = movie.title
  desc.text = movie.description

  let imageURL = URL(string: movie.image)
   image.kf.indicatorType = .activity
  image.kf.setImage(with: imageURL)
}
```

- movie의 데이터 변화를 View에 바로 적용하기 위해 `didSet`에 메소드 적용

- movie데이터를 sell 값에 할당. 영화 포스터 이미지 처리를 위해서는 `KignkFisher` 라이브러리를 사용했다.

# 결과

![](/images/tistory/65/03.gif)

- 데이터 결과의 갯수에 따라 알아서 Cell의 수를 계산해 `CollectionView`에 그려낸다. (애니메이션은 덤)

# Reference

[Mixing DiffableDataSource with UIKit and Combine  Part.1 : iOS Movies Catalogue](https://medium.com/dataseries/mixing-diffabledatasource-with-uikit-and-combine-part-1-ios-movies-catalogue-65cf90e37305)

[API for IMDb, TMDb, Wikipedia and more - IMDb-API](https://imdb-api.com/)

## 2026년에 다시 보며

이 글은 UIKit 화면 상태를 Combine 으로 묶는 방식이다. 새로 만든다면 iOS 17 의 Observation(`@Observable`)과 Swift Concurrency 를 먼저 검토할 것 같다.

UIKit 도 iOS 18 부터 `@Observable` 모델을 읽는 코드를 자동으로 추적해 화면을 갱신하는 기능을 Info.plist 설정으로 켤 수 있게 됐다. Combine 파이프라인이 꼭 필요한 곳(디바운스 · 여러 스트림 결합 등)만 남기고 나머지는 이쪽으로 옮기는 편이 단순하다.
