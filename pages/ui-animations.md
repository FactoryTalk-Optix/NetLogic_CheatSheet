# UI Animations

Animations allow variable values to be changed with a defined (or infinite) number of iterations; these animations can also be contained in parallel (run concurrently) or sequential animations.

## Number animation

Numerical animations allow the offset to be added to the value of the variable by defining an initial value to a final value by also defining the relevant type of interpolation (the value is an enumeration called EasingCurve)

```csharp
private NumberAnimation CreateNumberAnimation(IUANode ownerAnimationNode, NodeId targetAnimationNode, int from, int to, int loops, bool running, int duration, EasingCurve interpolation)
{
    // Look at the count of current animations in the node
    int animationCounter = ownerAnimationNode.GetNodesByType<NumberAnimation>().Count();
    // Create a new Number Animation with a browseName with index +1 over the current present animations
    var numberAnimation = InformationModel.Make<NumberAnimation>($"NumberAnimation{animationCounter + 1}");
    // Set animation parameters, target animation node must be a NodeId
    numberAnimation.Target = targetAnimationNode;
    numberAnimation.From = from;
    numberAnimation.To = to;
    numberAnimation.Loops = loops;
    numberAnimation.Running = running;
    numberAnimation.Duration = duration;
    numberAnimation.EasingCurve = interpolation;
    // Add the animation node to the target node
    ownerAnimationNode.Add(numberAnimation);
    // Return the new animation for further future use
    return numberAnimation;
}
```

## Sequential animation

Sequential animation allows all animations of this object to be performed sequentially

```csharp
private SequentialAnimation CreateSequentialAnimation(IUANode ownerAnimationNode, int loops, bool running)
{
    // Look at the count of current animations in the node
    int animationCounter = ownerAnimationNode.GetNodesByType<SequentialAnimation>().Count();
    // Create a new Sequential Animation with a browseName with index +1 over the current present animations
    var sequentialAnimation = InformationModel.Make<SequentialAnimation>($"SequentialAnimation{animationCounter + 1}");
    // Set animation parameters
    sequentialAnimation.Loops = loops;
    sequentialAnimation.Running = running;
    // Add the animation node to the target node
    ownerAnimationNode.Add(sequentialAnimation);
    // Return the new animation for further future use
    return sequentialAnimation;
}
```

## Parallel animation

Parallel animations allow all animations of this object to run simultaneously

```csharp
private ParallelAnimation CreateParallelAnimation(IUANode ownerAnimationNode, int loops, bool running)
{
    // Look at the count of current animations in the node
    int animationCounter = ownerAnimationNode.GetNodesByType<ParallelAnimation>().Count();
    // Create a new Parallel Animation with a browseName with index +1 over the current present animations
    var parallelAnimation = InformationModel.Make<ParallelAnimation>($"ParallelAnimation{animationCounter + 1}");
    // Set animation parameters
    parallelAnimation.Loops = loops;
    parallelAnimation.Running = running;
    // Add the animation node to the target node
    ownerAnimationNode.Add(parallelAnimation);
    // Return the new animation for further future use
    return parallelAnimation;
}
```

### Example of creating animations on a TextBox

```csharp
[ExportMethod]
public void GenerateStuffAnimation()
{
    var targetUiObject = Project.Current.Get<Item>("UI/Screens/Screen1/TextBox1");
    var parallelAnimation = CreateParallelAnimation(targetUiObject, 10, true);
    _ = CreateNumberAnimation(parallelAnimation, targetUiObject.OpacityVariable.NodeId, 0, 100, -1, true, 2000, EasingCurve.Linear);
    var sequentialAnimation = CreateSequentialAnimation(targetUiObject, 5, true);
    _ = CreateNumberAnimation(sequentialAnimation, targetUiObject.RotationVariable.NodeId, 0, 360, -1, true, 2000, EasingCurve.Linear);

}
```

## Value change animation

Animation on value change allows you to define the animation and duration to make the new value reach the value that is set in the variable

```csharp
[ExportMethod]
public void GenerateStuffBehaviorAnimation()
{
    var targetUiObject = Project.Current.Get<Item>("UI/Screens/Screen1/TextBox1");
    CreateBehaviorAnimation(targetUiObject.WidthVariable, 5000, EasingCurve.InOutQuad);
}

private void CreateBehaviorAnimation(IUANode targetAnimationNode, int duration, EasingCurve interpolation)
{
    // Check that targetNode is a variable and does not already have Behavior Animation
    if (targetAnimationNode is not IUAVariable || targetAnimationNode.GetNodesByType<BehaviourAnimation>().Any())
    {
        return;
    }
    // Create new Behavior Animation
    var behaviorAnimation = InformationModel.Make<BehaviourAnimation>($"Value change animation");
    // Set animation parameters
    behaviorAnimation.Duration = duration;
    behaviorAnimation.EasingCurve = interpolation;
    // Add the animation node to the target node
    targetAnimationNode.Add(behaviorAnimation);
}
```
