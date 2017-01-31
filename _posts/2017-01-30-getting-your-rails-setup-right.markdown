---
layout: post
title:  "Getting It Right - Rails Setup"
date:   2017-01-30 11:39 AM
categories: ruby-on-rails
author: Mingsheng Gan
---

Setting up your Rails project correctly is of utmost importance.  Obvious setup instructions somehow gets missed, and entire teams suffer from the repercussions of a bad setup.

This post describes some common common configuration mistakes specific to the `test-suite` that we have rectified. Simple but essential, we hope that this article helps you improve your codebase and enhance your developmental workflow.

# 3 Common Setup Mistakes

Learning to observe and having zero tolerance for obvious repetitions go a long way. Most popular gems typically consist of a configuration to simplify its usage, and we should always keep that in mind.

## Symptom: FactoryGirls thrown all over the place

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

## Symptom: Seeing RSpec in every file
Do you need to require rails_helper in every file? Or, type `RSpec` before every `describe`?
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

## Symptom: Why am I typing `spring` everytime?

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

```
# After: spring is in the closet now
~/my/project/$ rspec ./app/helpers/repetition_helper_spec.rb
```

# Conclusion

Learning to recognize these easy fixes easily simplifies your codebase significantly. Thank you for reading the article and we hope it helps you improve the quality of your codebase.

[configure-factorygirl][http://www.rubydoc.info/gems/factory_girl/file/GETTING_STARTED.md#Configure_your_test_suite]
[configure-rspec][https://www.relishapp.com/rspec/rspec-rails/docs/upgrade#default-helper-files]
[configure-spring][https://github.com/rails/spring#setup]
