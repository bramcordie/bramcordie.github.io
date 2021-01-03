---
title: Reviewing my first Symfony2 project
date: 2014-09-21 14:16:12
tags: [Symfony, PHP]
---

## History
In 2012 I built my first Symfony2 project. The most recent version
 back then was 2.0.12 which I figured out from the now deprecated _/deps_
 file.

```
[symfony]
    git=http://github.com/symfony/symfony.git
    version=v2.0.12
```

This project was part of my internship where I was given a LAMP stack to deploy
 on. I chose Symfony because of the resemblance with __ASP.NET MVC__ which I used
 on a previous project and enjoyed working with.

## Deployment
I used SVN before but I wasn't familiar with GIT. The project was under version
 control but the only version I found is the zip file I handed over at the
 end of my internship.

If this project requires further development, it would be a good idea to put
 it under version control. The vendor folder can be excluded together with
 the nbproject that somehow snuck into the zip file. Because _2.0_ lacks
 composer, the command to install vendors is different from what I'm used to
 but the workflow stays the same: `php bin/vendors install`

I had to download the missing _.gitignore_ file from the symfony-standard
 repository. After running `git init; git add .` in the root of the project,
 further changes can be tracked.

After changing the _/app/config/parameters.ini_ file, creating the database
 and running `php app/console doctrine:schema:update --force`, the project is
 available in my local development environment. The Apache and PHP config that I
 use for more recent projects seems to do the job, no tweaking required. Just to
 be sure, I checked _/web/config.php_, it did not complain.

## The Application
### Missing data
Clicking through the website I ran into the following errors:

- There are no questions!
- Can't find any question catergories!

Other pages just show a blank area where a list of options should appear. For
 example: when starting an assessment you should be able to pick an _opleiding_
 but none are presented.

There are 2 measures we can take to counter these issues.

First of all, showing sensible messages when data is missing to properly render
 views. Depending on the role of the user and their ability to add content, you
 can point them in the right direction and guide them through their first
 encounter with the application.

A second solution is seeding the project with the help of Doctrine fixtures.
 These fixtures can be run on command so they are extremely helpful if you mess
 up your data or you quickly need the dummy content.

### Bulky controllers
The controllers in this project ended up being quite bulky. Looking back, they
 provide plenty of examples where improvements can be made.

#### Form classes
Instead of defining forms inside the controller they can be defined in separate
 form classes that can directly map to existing model entities or view models.
 This way we can consolidate duplicate code in controllers and compose more
 elaborate forms by combining different form classes.

#### Repositories
Another way of keeping controllers dry is creating custom repository classes to
 persist and retrieve entities.

The following controller action code looks for all assessments created between
 a begin and end date.

```php
<?php
// src/OAT/OATBundle/Controller/StatisticsController.php

public function assessmentAction()
{
  ...
  $categoryAssessments = $this->getDoctrine()
    ->getRepository('OATBundle:CategoryAssessment')
    ->createQueryBuilder('c')
    ->where("c.status = 1")
    ->andWhere("c.created <= :endDate")
    ->andWhere("c.created >= :startDate")
    ->setParameters(array('startDate' => $startDate, 'endDate' => $endDate))
    ->getQuery()
    ->getResult();
  ...
}
```

This Query can be moved to a repository class where we define it as a function.

```php
<?php
// src/OAT/OATBundle/Entity/AssessmentRepository.php
class AssessmentRepository extends EntityRepository
{
    public function findAllByTimespan($startDate, $endDate){
        ...
    }
}
```

We can then replace the query in the controller action with the repository
 function.

```php
<?php
// src/OAT/OATBundle/Controller/StatisticsController.php
public function assessmentAction()
{
  ...
  $categoryAssessments = $this->getDoctrine()
    ->getRepository('OATBundle:CategoryAssessment')
    ->findAllByTimespan($startDate, $endDate);
  ...
}
```

#### Keep domain logic inside the model
The following controller action code takes a list of category groups, which
 can contain overlapping categories, matches them against a list of all the
 existing categories and builds a set with unique categories to create an
 assessment.

```php
<?php
// src/OAT/OATBundle/Controller/AssessmentController.php newAction
$categories = Array();

$categoryGroups = $this->getDoctrine()
  ->getRepository('OATBundle:QuestionCategoryGroup')
  ->findAll();

$selectedGroups = Array();

foreach($categoryGroups as $categoryGroup)
{
  if($request->request->get('category-group-'.$categoryGroup->getId())) {
    $selectedGroups[] = $categoryGroup;
    foreach($categoryGroup->getQuestionCategoryGroupMember() as $category)
    {
      if(!in_array($category->getQuestionCategory(), $categories)) {
          $categories[] = $category->getQuestionCategory();
      }
    }
  }
}
```

You don't want the repository call and http request logic to end up in the model.
 The following code will match the requested group IDs against existing
 category groups.

```php
<?php
// src/OAT/OATBundle/Controller/AssessmentController.php
private function getRequestedCategoryGroupIds(Request $request) {
  // Get all the posted variables
  $postedVariables = $request->request->all();

  // There's a post variable prefixed with 'category-group-' per category group.
  // You can't filter on key so the array must be flipped.
  $postedCategoryGroups = array_filter(
      array_flip($postedVariables),
      function($key) {
        return strpos($key, 'category-group-') === 0;
      }
  );

  // Strip the group IDs
  $categoryGroupIds = array_map(function($value) {
    return intval(str_replace("category-group-", "", $value));
  }, $postedCategoryGroups);

  return $categoryGroupIds;
}

public function newAction(Request $request) {
  $categoryGroupIds = $this->getRequestedCategoryGroupIds($request);

  // Find the matching group entities
  $categoryGroups = $this->getDoctrine()
    ->getRepository('OATBundle:QuestionCategoryGroup')
    ->findById($categoryGroupIds);
  ...
}

```

Further down in the action, the category groups are added after the assessment
 object is created. Instead of just passing the groups to the constructor, all
 their categories were reduced to a set to use as parameter.

```php
<?php
// src/OAT/OATBundle/Controller/AssessmentController.php
$assessment = new Assessment($categories);
$assessment->setCreated(new DateTime("now"));
foreach($selectedGroups as $group)
{
  $assessment->addQuestionCategoryGroup($group);
}
```

You should be able to create an assessment by passing the groups and let
 the constructor deal with them. The model should decide if duplicate categories
 are allowed without involving the controller for any domain logic.

```php
<?php
// src/OAT/OATBundle/Entity/Assessment.php
/**
 * @param QuestionCategoryGroup[] $groups
 */
public function __construct(Array $groups)
{
  ...
  foreach ($groups as $group) {
    $this->addQuestionCategoryGroup($group);
  }
  ...
}

public function addQuestionCategoryGroup(QuestionCategoryGroup $group)
{
  $this->questionCategoryGroups[] = $group;

  foreach($group->getQuestionCategoryGroupMembers() as $categoryGroupMember) {
    $this->addQuestionCategory($categoryGroupMember->getQuestionCategory());
  }
}

public function addQuestionCategory(QuestionCategory $category)
{
  if(!in_array($this->getQuestionCategories(), $category) {
    $this->questionCategories[] = $category;
  };
}
```

### Managing Dependencies
#### Back-end
Switching to __Composer__ would make it easier to manage security updates. This
 requires updating Symfony to at least _v2.1_. Going for _v2.3_ will get you
 long-term support and makes sure you are only running stable versions of external
 dependencies. This project does not rely heavily on third-party bundles so upgrading
 should not be that hard by following the [Symfony upgrade guide](https://github.com/symfony/symfony/blob/master/UPGRADE-2.1.md).

#### Front-end
Javascript libraries are stored in the OAT bundle. This creates unneeded bloat
 in the repository and requires you to downloaded and add them manually.

 For more recent projects I'm using [Bower](http://bower.io/) and it makes
 developing with small javascript libraries so much easier. You can copy over a
 list of dependencies from existing projects and running `bower install` will
 get the locally cached files or download them when needed. Assetic can still be
 used to minify everything to a single file.