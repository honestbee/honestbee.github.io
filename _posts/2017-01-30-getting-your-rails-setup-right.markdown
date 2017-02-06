---
layout: post
title:  "Getting It Right - Rails Setup"
date:   2017-01-30 11:39 AM
categories: ruby-on-rails
author: Mingsheng Gan
---

Setting up your Rails project correctly is of utmost importance. Obvious setup instructions somehow get missed, and entire teams suffer from the repercussions of a bad setup.

This post describes some common configuration mistakes specific to the `test-suite` that we have rectified. Simple but effective, this article points out several high-leverage fixes which help you improve your codebase and enhance your developmental workflow.

## 3 Common Setup Mistakes

Having zero tolerance for obvious repetitions go a long way. Most popular gems typically consist of a configuration to simplify its usage and we should always keep that in mind. Here are 3 obvious setup mistakes that we have corrected:

### Symptom: FactoryGirls littered all over the place

Have you seen such a codebase?
```ruby
# Before: She's everywhere!
bad_codebase = FactoryGirl.create(:codebase)
better_codebase = FactoryGirl.create(:codebase, :with_traits)
```

Correction: [Configure FactoryGirl properly][configure-factorygirl]
```ruby
# After: She's in the closet now
clean_codebase = create(:codebase)
better_codebase = create(:codebase, :with_traits)
```

Keywords such as `create`, `build` or `build_stubbed` are typically used in almost every test case. Measuring the number of lines of codes affected, a single-line fix like the above affects `416 / 7413` lines of code across `76 files` in the spec suite, improving 5% of the codebase with just 1 line of code change.

### Symptom: RSpec required specifically in every spec file

Do you need to `require 'rails_helper'` in every spec file? Or, type `RSpec` before every `describe`?
```ruby
# Before: This helper turns up in every room
require 'rails_helper'

RSpec.describe Excellence, type: :helper do
  ...
end
```

Correction: [Require rails_helper instead of spec_helper][configure-rspec]
```
# /path/to/project/.rspec
--require rails_helper # instead of spec_helper
```

```ruby
# After: She does her job but from within the closet
describe Excellence, type: :helper do
  ...
end
```

A change which reduces `~3 LOCs across almost each of 97 spec files`. In terms of magnitude of change, this is smaller. Doubtlessly though, such changes are essential, and in our opinion, symbolic in showing how we are taking code quality seriously.

### Symptom: Having to type `spring` everytime

We use `spring` to speed up our development workflow. To support `rspec`, we use `spring-commands-rspec`.
```
# Before: Repetition occurs even in console mode
~/my/project/$ spring rspec ./app/helpers/repetition_helper_spec.rb
```

Question yourself, [would spring gem developers not have thought of it?][configure-spring]
```
$ bundle install
$ bundle exec spring binstub --all
```

```diff
# After: spring is in the closet now
- ~/my/project/$ spring rspec ./app/helpers/repetition_helper_spec.rb
+ ~/my/project/$ rspec ./app/helpers/repetition_helper_spec.rb
```

Enabling spring by default introduces `evil side-effects`. Herein, spring potentially causes specs to fail until spring gets restarted (as it helps us avoid re-running the server setup) which may cause developers precious time solving a non-existent problem. That is a big risk to take!

However, weighing it against junior developers unknowingly losing 5 seconds each time running rspec on a single file, we found it worthwhile. Not only does it speed up the developmental process, having spring automatically installed also `ensures that developers execute tests more often` (which indirectly translates to more tests being written and tests being executed more often).

Some basic calculation suggests that a time-saving of `5 seconds across 50 developers` implies 250 seconds is lost on every test execution. With tests executed, say 5 times per hour, this translates to `2.78 hours lost every work-day`, or `55.55 hours lost every month`. That's `approximately 2+ days` for a single developer. May not be much, but we get happier developers at the same time!

## Conclusion

Usage-specific tweaks are typically available in gems to simplify your codebase. Learning to recognize these easy fixes easily DRYs up your codebase significantly. Thank you for reading the article and we hope it helps you improve the quality of your codebase.

[configure-factorygirl]: http://www.rubydoc.info/gems/factory_girl/file/GETTING_STARTED.md#Configure_your_test_suite
[configure-rspec]: https://www.relishapp.com/rspec/rspec-rails/docs/upgrade#default-helper-files
[configure-spring]: https://github.com/rails/spring#setup
