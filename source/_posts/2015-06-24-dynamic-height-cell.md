layout: post
date: 2015-06-24 20:28:12
title: 可变高度的UITableViewCell
tags: 
- iOS
---

可变高度的`UITableViewCell`用到的很多，简单总结一下利用autolayout实现自动计算高度的`UITableViewCell`的要点。

首先，设置Cell内部subviews的autolayout约束，这里就不再阐述。对于布局比较复杂的Cell，通过代码设置约束还是灵活很多的。

`UITableView`在`UITableViewDelegate Protocol`的如下方法中计算Cell的高度，所以我们也主要在这个方法中计算高度的值。
```objc
- (CGFloat)tableView:(UITableView )tableView heightForRowAtIndexPath:(NSIndexPath )indexPath;
```
1. 为了利用autolayout计算高度，我们先得有一个Cell的实例。可以创建一个Cell的成员变量，或者静态变量。
2. 之后调用`cell.contentView`的`setNeedsLayout`和`layoutIfNeeded`方法触发autolayout计算合适的布局。
3. 最后调用`cell.contentView`的`systemLayoutSizeFittingSize:`方法得到`cell.contentView`合适的高度。
4. `UITableViewCell`的高度是`contentView`的高度加上分割线的高度(通常是1)，所以返回`contentView高度 + 1`才是正确的高度。

相应代码如下：
```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath { 
    static CustomCell *offscreenCell; 
    if (!offscreenCell) { 
        offscreenCell = [[CustomCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"Cell"]; 
    } 
    // configure offscreenCell ... 
    [offscreenCell.contentView setNeedsLayout]; 
    [offscreenCell.contentView layoutIfNeeded]; 
    CGFloat height = [offscreenCell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height; 
    return height + 1; 
}
```
同时，需要保证：`cell.contentView.translatesAutoresizingMaskIntoConstraints = NO;`
如果Cell包含支持多行文本的UILabel，还要设置
```objc
cell.textLabel.lineBreakMode = NSLineBreakByWordWrapping;
cell.textLabel.numberOfLines = 0;
```
这里有一个效率问题，`UITableView`是一次性计算所有Cell的高度，所以对于Cell比较多的情况需要较长时间计算才能显示内容。这个问题可以通过实现`- (CGFloat)tableView:(UITableView )tableView estimatedHeightForRowAtIndexPath:(NSIndexPath )indexPath;`方法解决。首先通过该方法返回一个估计值，之后要显示Cell的时候才计算准确的高度。

如果Cell里包含`UITextView`，那么还需要调用`UITextView`的`sizeThatFit:`方法进行额外的处理，示例代码如下：
```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    static CustomCell *offscreenCell; 
    if (!offscreenCell) { 
        offscreenCell = [[CustomCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:@"Cell"]; 
    } 
    // configure offscreenCell ... 
    [offscreenCell.contentView setNeedsLayout]; 
    [offscreenCell.contentView layoutIfNeeded]; 
    CGSize size = [offscreenCell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize];
    CGSize textViewSize = [offscreen.textView sizeThatFits:CGSizeMake(cell.textView.frame.size.width, FLT_MAX)];
    CGFloat h = size.height + textViewSize.height;
    h = h > MIN_CELL_HEIGHT ? h : MIN_CELL_HEIGHT;  //MIN_CELL_HEIGHT是CELL显示的最低高度
    return 1 + h;
}
```