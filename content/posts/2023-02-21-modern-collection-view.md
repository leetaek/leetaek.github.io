+++
title = "[UIKit]Modern Collection View"
date = 2023-02-21T15:31:13+09:00
draft = false
tags = ["ios", "swift", "ui", "uikit"]
categories = ["iOS"]
tistory = "https://leetaek.tistory.com/63"
+++

# Collection View

## UICollectionViewDelegate, UICollectionViewDataSource

```swift
class ViewController: UIViewController {
  @IBOutlet weak var collectionView: UICollectionView!

  override func viewDidLoad() {
    super.viewDidLoad()

    collectionView.delegate = self
    collectionView.dataSource = self
    collectionView.register(UINib(nibName: "CollectionViewCell", bundle: nil), forCellWithReuseIdentifier: "CollectionViewCell")
  }
}

// MARK: - 기존 CollectionView를 사용하는 방법
extension ViewController: UICollectionViewDelegate, UICollectionViewDataSource {
  func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
    return 10
  }

  func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
    guard let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "CollectionViewCell", for: indexPath) as? CollectionViewCell else {
      return UICollectionViewCell()
    }

    cell.label.text = indexPath.row.description
    return cell
  }
}

extension ViewController: UICollectionViewDelegateFlowLayout {
  func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize {

    let itemSpacing: CGFloat = 10
    let width: CGFloat = (collectionView.bounds.width - itemSpacing) / 2
    let height: CGFloat = width * 10/7

    return CGSize(width: width, height: height)
  }
}
```

# Modern Collection View (iOS 13.0+)

![](/images/tistory/63/01.webp)

![](/images/tistory/63/02.webp)

![](/images/tistory/63/03.webp)

*문제가 되는 상황과 케이스*

- 기존의 `collectionView` 형태에서는 UI와 `DataSource`의 버전(`truth`)가 같지 않을때 멈추는 문제가 있었다. `reloadData()`로 해결할 수 있지만, 그럴 경우 애니메이션이 씹히는 현생이 발생. 이를 해결하기 위해 `snapShot`이라는 새로운 개념을 접근하는 것이 개요.

- `diffable`은 화면에 보여질 아이템이 업데이트 될 때 마다 collectionView가 자동으로 업데이트 된다는 의미. **이전 데이터와의 차이를 계산하고 애니메이션을 통해 보여준다.**

## UICollectionViewDiffableDataSource

```swift
func setUpDataSource() {
   dataSource = UICollectionViewDiffableDataSource<Int, Int>(collectionView: collectionView) { (collectionView, indexPath, cellNumber) -> UICollectionViewCell? in
     guard let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "CollectionViewCell", for: indexPath) as? CollectionViewCell else {
       return UICollectionViewCell()
     }

     cell.label.text = cellNumber.description

     return cell
   }
 }

func performCell() {
   let cellNum: [Int] = (0..<10).map{ Int($0) }

   var snapShot = NSDiffableDataSourceSnapshot<Int, Int>()
   snapShot.appendSections([0])
   snapShot.appendItems(cellNum)
   self.dataSource.apply(snapShot)
 }
```

# UICollectionViewLayout

## UICollectionViewFlowLayout

- 기존의 방법대로

## UICollectionViewCompositionalLayout (iOS 13.0 +)

- [UICollectionViewCompositionalLayout (iOS 13.0 +)](https://leetaek.tistory.com/61)

# 전체 코드

```swift
class ViewController: UIViewController {

  @IBOutlet weak var collectionView: UICollectionView!
  var dataSource: UICollectionViewDiffableDataSource<Int, Int>!

  override func viewDidLoad() {
    super.viewDidLoad()

//    collectionView.delegate = self
//    collectionView.dataSource = self
    collectionView.register(UINib(nibName: "CollectionViewCell", bundle: nil), forCellWithReuseIdentifier: "CollectionViewCell")

    // UICollectionViewDiffableDataSource를 사용한 방법
    collectionView.collectionViewLayout = createLayout()
    setUpDataSource()
    performCell()

  }
}

 // MARK: - 기존 CollectionView를 사용하는 방법
extension ViewController: UICollectionViewDelegate, UICollectionViewDataSource {
  func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
    return 10
  }

  func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
    guard let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "CollectionViewCell", for: indexPath) as? CollectionViewCell else {
      return UICollectionViewCell()
    }

    cell.label.text = indexPath.row.description
    return cell
  }
}

extension ViewController: UICollectionViewDelegateFlowLayout {
  func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize {
    let itemSpacing: CGFloat = 10
    let width: CGFloat = (collectionView.bounds.width - itemSpacing) / 2
    let height: CGFloat = width * 10/7

    return CGSize(width: width, height: height)
  }
}

// MARK: - UICollectionViewDiffableDataSource을 사용한 방법
extension ViewController {
  func setUpDataSource() {
     dataSource = UICollectionViewDiffableDataSource<Int, Int>(collectionView: collectionView) { (collectionView, indexPath, cellNumber) -> UICollectionViewCell? in
       guard let cell = collectionView.dequeueReusableCell(withReuseIdentifier: "CollectionViewCell", for: indexPath) as? CollectionViewCell else {
         return UICollectionViewCell()
       }

       cell.label.text = cellNumber.description

       return cell
     }
   }

  func performCell() {
     let cellNum: [Int] = (0..<10).map{ Int($0) }

     var snapShot = NSDiffableDataSourceSnapshot<Int, Int>()
     snapShot.appendSections([0])
     snapShot.appendItems(cellNum)
     self.dataSource.apply(snapShot)
   }

  func createLayout() -> UICollectionViewLayout {
    let layout: UICollectionViewCompositionalLayout = {
      let itemSpacing: CGFloat = 10
      let width: CGFloat = 1.0 / 2.0
      let height: CGFloat = 1.0 / 2.5

      // item
      let itemSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(width),
        heightDimension: .fractionalHeight(1)
      )
      let item = NSCollectionLayoutItem(layoutSize: itemSize)
      item.contentInsets = NSDirectionalEdgeInsets(top: itemSpacing, leading: itemSpacing, bottom: itemSpacing, trailing: itemSpacing)

      // group
      let groupSize = NSCollectionLayoutSize(
        widthDimension: .fractionalWidth(1),
        heightDimension: .fractionalHeight(height)
      )
      let group = NSCollectionLayoutGroup.horizontal(layoutSize: groupSize, subitems: [item])

      //section
      let section = NSCollectionLayoutSection(group: group)
      section.contentInsets = NSDirectionalEdgeInsets(top: itemSpacing, leading: itemSpacing, bottom: itemSpacing, trailing: itemSpacing)

      return UICollectionViewCompositionalLayout(section: section)
    }()

    return layout
  }
}
```

# Reference

- Documents

[Apple Developer Documentation](https://developer.apple.com/documentation/uikit/views_and_controls/collection_views/implementing_modern_collection_views)

- collectionView layout

[[iOS] UICollectionView와 관련 된 경험들](https://sueaty.tistory.com/187)

- Modern Collection View

[Diffable Datasource](https://zeddios.tistory.com/1197)

[[iOS] Modern Collection View - List 구현](https://leechamin.tistory.com/555)

- UICollectionViewLayout

[[iOS - swift] 1. UICollectionViewCompositionalLayout - 개념 (section, group, item)](https://ios-development.tistory.com/945)

## 2026년에 다시 보며

iOS 14 부터는 셀 등록도 `UICollectionView.CellRegistration` 으로 바뀌어서, 이 글의 `register(UINib…)` + 문자열 식별자 + 캐스팅 조합을 쓰지 않아도 된다.

```swift
let registration = UICollectionView.CellRegistration<CollectionViewCell, Int> { cell, _, number in
    cell.label.text = number.description
}
dataSource = UICollectionViewDiffableDataSource<Int, Int>(collectionView: collectionView) { collectionView, indexPath, number in
    collectionView.dequeueConfiguredReusableCell(using: registration, for: indexPath, item: number)
}
```

목록 형태라면 `UICollectionLayoutListConfiguration` 으로 레이아웃을 따로 짜지 않아도 되고, iOS 15 의 `reconfigureItems` 를 쓰면 셀을 다시 만들지 않고 내용만 갱신할 수 있다.
