![MANUela Logo](./images/logo.png)

# Code Change  <!-- omit in toc -->
This demo modules show how to implement a code change to manuela component. The story is that in the code processing temperature data, there is an unnecessary conversion from celsius to fahrenheit that needs to be removed.

- [Prerequisites](#prerequisites)
- [Demo Preparation](#demo-preparation)
  - [Code Change Prep](#code-change-prep)
- [Demo Execution](#demo-execution)
  - [Step 1: Show the bug](#step-1-show-the-bug)
  - [Step 2: Fix the bug](#step-2-fix-the-bug)
  - [Step 3: Commit and push changes](#step-3-commit-and-push-changes)

## Prerequisites


The management hub and factory edge clusters are up and running, sensor data is flowing into the IoT dashboard.
You forked and cloned manuela-dev on your local machine.

## Demo Preparation

You will probably want to run through the ci-cd demo right after this module. Be sure to review [its preparation](module-ci-cd-pipeline.md#Demo-preparation) steps as well.

### Code Change Prep

Check that the source code we are going to remove in manuela-dev/components/iot-consumer/index.js, line ~193  is NOT commented out and looks like this:
![image alt text](images/crw_4.png)

In case the code got lost because someone deleted it instead of commenting out, here it is for easy cut'n'paste:

```javascript
    var modifiedValue = (Number(elements[2]) * 9/5) + 32;
    var newData = data.replace(elements[2], modifiedValue);
    message = Buffer.from(newData, 'utf8');
```

In case the code was commented out, uncomment it so that you inject the failure. Deplopy the change with the CI/CD Pipeline module


## Demo Execution
Perform the demo with the following steps.


### Step 1: Show the bug
Open the iot-frontend in the manuela-tst environment, check that the bug is visible there. The temperatures values are way to high, alerts firing all the time:  
![image alt text](images/crw_5.png)

### Step 2: Fix the bug
Fix the bug by adding the comments, save the changes.  
![image alt text](images/crw_6.png)

As code snippet:
```javascript
/*
    var modifiedValue = (Number(elements[2]) * 9/5) + 32;
    var newData = data.replace(elements[2], modifiedValue);
    message = Buffer.from(newData, 'utf8');
*/
```

### Step 3: Commit and push changes

```
cd ~/manuela-dev/components/iot-consumer/
git add index.js
git commit -m "fix F-C"
git push
```

Continue with the CI/CD Pipeline module for deploying the change.

