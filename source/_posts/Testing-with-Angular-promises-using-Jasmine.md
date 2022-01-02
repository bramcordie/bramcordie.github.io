---
title: Testing with Angular promises using Jasmine
date: 2015-11-3 19:37:27
tags: [Angular, Jasmine, Javascript]
---

When dealing with promises while writing Jasmine tests for an Angular project there are several approaches you can take.

We will start with some specs that just have to consume a promise. You
usually don’t care what the dependency that creates the promise does. If this is the case, create a spy for it.

The following tests for a membership viewer will try to get a list of members from a membership service. The service might need to run some asynchronous code to retrieve the list so it returns a promise instead of the list itself.

```javascript
describe('Membership Viewer', function () {
  var membershipViewer, membershipService;
  
  beforeEach(function () {
    membershipService = jasmine.createSpyObj('membershipService', ['getMembers']);
    membershipViewer = new MembershipViewer(membershipService);
  });
  
  it('should display a list of group members', function() {
    var members = ['Alice', 'Bob'];
    membershipService.getMembers.and.returnValue($q.resolve(members));
    
    membershipViewer.showMembers('a-group-id');
    $scope.$apply();
    expect(membershipService.getMembers).toHaveBeenCalledWith('a-group-id');
    expect(membershipViewer.members).toEqual(members);
  });

  it('should dislay an error when unable to retrieve group members', function () {
    var asyncError = "Unknown group id!";
    membershipService.getMembers.and.returnValue($q.reject(asyncError));
  
    membershipViewer.showMembers('unknown-group-id');
    $scope.$apply();
    expect(membershipService.getMembers.toHaveBeenCalledWith('unknown-group-id');
    expect(membershipViewer.error).toEqual(asyncError);
  });
})
```

Remember that we are only testing the viewer, it does not matter how members are retrieved. We immediately resolve the members promise and simply check if the service was called with the expected parameters.

You might notice that we call an apply every time we expect something returned from the service. This is because a digest is required to resolve the promises even if we return the results right away.

If you really need to track the whole lifecycle of a promise you can make use of a deferred. When you defer a result yourself, you have to make sure your test actually runs up to the point where everything is resolved.

Let’s say you want to indicate that members are being loaded while you wait for the membership service to return a result. The loading state is tracked by a variable called loadingMembers. You now need to take control of the members promise to check that intermediate loading state.

```javascript
it('should indicate that members are loading while waiting for results', function (done){
  var deferredMembers = $q.defer();
  var membersPromise = deferredMembers.promise;
  var assertMembersLoaded = function () {
    expect(membershipViewer.members).toEqual(['Alice', 'Bob']]);
    expect(membershipViewer.loadingMembers).toEqual(false);
    done();
  };
  membershipService.getMembers.and.returnValue(membersPromise);

  membershipViewer.showMembers('a-group-id');
  expect(membershipViewer.loadingMembers).toEqual(true);

  membersPromise.then(assertMembersLoaded);
  deferredMembers.resolve(['Alice', 'Bob']);
  $scope.$apply();
});
```

Jasmine provides a done function to help out with testing asynchronous behaviour. You pass it along with a specs’ closure and call it when you are sure everything is done. This way we know that all our expectations are met.

There’s an additional set of [third-party Jasmine matchers](https://github.com/bvaughn/jasmine-promise-matchers) dedicated to promises. I’d rather go easy on the matchers and just use a simple equal-to when possible. They do call a digest for you but at the same time you lose some control.