# CollectionViewPageEnable
通过继承UICollectionViewFlowLayout，实现UICollectionView的pageEnable功能

最近项目有个功能是照片浏览器，当预览照片时，可以整屏左右滑动，一想到整屏滑动，自然而然就想到了scrollview的pageEnable属性，于是就兴致勃勃的把该属性设为YES，可是显示出来的效果并不尽如人意，cell和cell滚动之后总会有一定单位的偏差，于是又想到了另外一个直接设置scrollView的偏移量的做法，然并卵。

![直接设置pageEnable属性为YES后的效果](https://upload-images.jianshu.io/upload_images/2172432-8385c3b38440b7c8.gif?imageMogr2/auto-orient/strip)


 在网上搜索解决办法，无意戳进了[链接](https://www.jianshu.com/p/8de72cb2aae5)，在通读完整篇文章后，我对评论区中提到的解决办法：**重写flowLayout里的targetContentOffsetForProposedContentOffset这个代理方法**来实现该效果比较感兴趣，于是就依照此原理进行了编写。

因为苹果系统已经提供给我们了一个流水布局，所以新建一个类继承于UICollectionViewFlowLayout。

 接下来需要重写父类中的一些方法，自定义UICollectionViewLayout时常用的方法有如下一些
```
//预布局方法 所有的布局应该写在这里面
- (void)prepareLayout
 
//此方法应该返回当前屏幕正在显示的视图（cell 头尾视图）的布局属性集合（UICollectionViewLayoutAttributes 对象集合）
- (NSArray *)layoutAttributesForElementsInRect:(CGRect)rect
 
//根据indexPath去对应的UICollectionViewLayoutAttributes  这个是取值的，要重写，在移动删除的时候系统会调用改方法重新去UICollectionViewLayoutAttributes然后布局
- (UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath
- (UICollectionViewLayoutAttributes *)layoutAttributesForSupplementaryViewOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)indexPath
 
//返回当前的ContentSize
- (CGSize)collectionViewContentSize
//是否重新布局 
- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds
 
//这4个方法用来处理插入、删除和移动cell时的一些动画 瀑布流代码详解
- (void)prepareForCollectionViewUpdates:(NSArray *)updateItems
- (UICollectionViewLayoutAttributes*)initialLayoutAttributesForAppearingItemAtIndexPath:(NSIndexPath *)itemIndexPath
- (nullable UICollectionViewLayoutAttributes *)finalLayoutAttributesForDisappearingItemAtIndexPath:(NSIndexPath *)itemIndexPath
- (void)finalizeCollectionViewUpdates
//9.0之后处理移动相关
- (UICollectionViewLayoutInvalidationContext *)invalidationContextForInteractivelyMovingItems:(NSArray<nsindexpath *> *)targetIndexPaths withTargetPosition:(CGPoint)targetPosition previousIndexPaths:(NSArray<nsindexpath *> *)previousIndexPaths previousPosition:(CGPoint)previousPosition NS_AVAILABLE_IOS(9_0)
- (UICollectionViewLayoutInvalidationContext *)invalidationContextForEndingInteractiveMovementOfItemsToFinalIndexPaths:(NSArray<nsindexpath *> *)indexPaths previousIndexPaths:(NSArray<nsindexpath *> *)previousIndexPaths movementCancelled:(BOOL)movementCancelled NS_AVAILABLE_IOS(9_0)</nsindexpath *></nsindexpath *></nsindexpath *></nsindexpath *>
```

我们选择重写这些方法中的4个方法即可完成需求
1.- (void)prepareLayout
```
- (void)prepareLayout {
    [super prepareLayout];
    // 设置滚动方向
    self.scrollDirection = UICollectionViewScrollDirectionHorizontal;
    // 设置collectionView的cell大小
    self.itemSize = CGSizeMake([UIScreen mainScreen].bounds.size.width, [UIScreen mainScreen].bounds.size.height);
}
```

2.- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds
```
- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds {
    return YES;
}
```

3.-(NSArray *)layoutAttributesForElementsInRect:(CGRect)rect
```
-(NSArray *)layoutAttributesForElementsInRect:(CGRect)rect {
    NSArray *array = [super layoutAttributesForElementsInRect:rect];
    
    return array;
}
```

4.- (CGPoint)targetContentOffsetForProposedContentOffset:(CGPoint)proposedContentOffset withScrollingVelocity:(CGPoint)velocity
```
- (CGPoint)targetContentOffsetForProposedContentOffset:(CGPoint)proposedContentOffset withScrollingVelocity:(CGPoint)velocity {
    
    //1. 获取UICollectionView停止的时候的可视范围
    CGRect rangeFrame;
    rangeFrame.size = self.collectionView.frame.size;
    rangeFrame.origin = proposedContentOffset;
    
    NSArray *array = [self layoutAttributesForElementsInRect:rangeFrame];
    
    //2. 计算在可视范围的距离中心线最近的Item
    CGFloat minCenterX = CGFLOAT_MAX;
    CGFloat collectionCenterX = proposedContentOffset.x + self.collectionView.frame.size.width * 0.5;
    for (UICollectionViewLayoutAttributes *attrs in array) {
        if(ABS(attrs.center.x - collectionCenterX) < ABS(minCenterX)){
            minCenterX = attrs.center.x - collectionCenterX;
        }
    }
    
    //3. 补回ContentOffset，则正好将Item居中显示
    CGFloat proposedX = proposedContentOffset.x + minCenterX;
    // 滑动一屏时的偏移量
    CGFloat mainScreenBounds = [UIScreen mainScreen].bounds.size.width + 10;
    // 正向滑动仅滑动一屏
    if (proposedX - self.lastProposedContentOffset >= mainScreenBounds) {
        proposedX = mainScreenBounds + self.lastProposedContentOffset;
    }
    // 反向滑动仅滑动一屏
    if (proposedX - self.lastProposedContentOffset <= -mainScreenBounds) {
        proposedX = -mainScreenBounds + self.lastProposedContentOffset;
    }
    
    self.lastProposedContentOffset = proposedX;
    
    CGPoint finialProposedContentOffset = CGPointMake(proposedX, proposedContentOffset.y);
    return finialProposedContentOffset;
}
```

![大功告成~](https://upload-images.jianshu.io/upload_images/2172432-a996282978cb90ac.gif?imageMogr2/auto-orient/strip)

如果你有好的意见或者建议，欢迎来[这里](https://www.jianshu.com/p/db9d31aba2e8)与我交流
