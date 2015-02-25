# Best Practices Really Really Rough Draft

## Logic in Tests

### Some basic ground rules:

#### Tests that change flow based on a flag parameter, to reuse code for multiple test scenarios: Not OK

##### Example:

    def test_wizard(back_and_forth=False):
        wizard = Wizard.launch()
        assert(…verify page state…)
        wizard.change_stuff()
        wizard.next()
        if back_and_forth:
            wizard.previous()
            assert(…verify state still there…)
            wizard next()
        assert(…verify page state…)
        wizard.change_stuff()
        wizard next()
        if back_and_forth:
            wizard.previous()
            assert(…verify data still there…)
            wizard next()
        assert(…verify page state…)
        wizard.change_stuff()
        # etc

###### Why Not?

1. Complexity obscures test flow. Can kind of tell what the back_and_forth version does, but not the regular version.

2. Combines two scenarios: advance through wizard, and go back and forth to check for data retention on each page. Not cohesive. Function = one intent.

This should be two different tests.

#### Tests with *value* parameters: OK (if your test harness can run them)

    def test_login(username, password, should_succeed):
        login = Login.launch()
        current_page = login.attempt_login(username, password)
        succeeded = current_page != login
        assert(succeeded == should_succeed, …)

Elsewhere:

    test_login('joe', 'user', should_succeed=True)
    test_login('bob', 'luser', should_succeed=False)

##### Why?

1. You need to do this for data-driven testing. It’s a standard practice.

2. The test is still easy to read. The verification is slightly more complex for needing to compare to a “should” value, but still pretty clear. In this case, we would take the hit because it’s what enables the test scenario to be reused for more than one test case.

3. Nothing about the test scenario is being changed. Function is still cohesive.

The values change, but the scenario remains the same: try `username` and `password`, verify expected success. The *test case* or *test data* (same thing—depends on your terminology) changes via parameterization, but the scenario is what a test function captures.

This one is simple, so if you decided to split it into `test_successful_login()` and `test_unsuccessful_login()`, that would also be reasonable. But for a larger test, it might be overkill and you may end up with a lot of flow code duplication just to save one comparison.

In practice, whether this is appropriate will depend on whether you're set up to do data-driven testing. But most test frameworks have some way to define a test this way and include a series of populated calls to it in the test file or elsewhere.

#### Tests with behavior parameters, where the logic is encapsulated in the app object: Can be OK

    def test_launch_calculator(method):
        calculator = Calculator.launch(method)
        assert(calculator is not None, …)
        
    @classmethod
    def Calculator.launch(cls, method=METHOD_SHORTCUT):
        if method == METHOD_SHORTCUT:
            type_keys(SHORTCUT_CALCULATOR)
        elif method == METHOD_MENU:
            select_menu(MENU_CALCULATOR)
        # etc.
        return Calculator()

    test_launch_calculator(METHOD_SHORTCUT)
    test_launch_calculator(METHOD_MENU)

##### Why?

1. Convenience. You’ll need a launch method anyway for the rest of the calculator tests, and it can be easier to encapsulate things like how you launch in it.

2. The test is still easy to read.

3. You still have single point of change via the `SHORTCUT_` and `MENU_` constants.

##### Why not?

You could argue it’s two different scenarios.

However, I would argue it’s a parameterized test of “Launch calculator via _method_”, with a method matrix of “shortcut” and “menu”.

That’s because the flow details of how you type in a shortcut or manipulate the menu aren’t important to the test, only the high level consideration that you used a shortcut or manipulated a menu.

But these cases are pretty rare. If the flow is important to the test, it should be in the test. If multiple flows are important, that’s multiple tests.

However, if you're finding that you're only splitting the test on behavior that's completely contained in the app object anyway, then moving the basis for the split into the app object might make sense.

### Tests that vary flow based on variations in the Application flow

This is the complex one.

Here is a case where the wizard is normally two pages, but sometimes there’s an optional page that pops up after the first one.

For the purposes of this example, assume for now that we don’t know when the page will pop up; it’s random, but there’s a consistent point where it’ll happen if it’s going to happen.

    def test_variable_wizard():
        vwizard = VWizard.launch()
        assert(…verify page…)
        vwizard.next()
        
    if vwizard.current_page == vwizard.OPTIONAL_PAGE:
        assert(…verify page…)
        vwizard.next()

    assert(…verify page…)
    vwizard.finish()

This is OK, given that it’s random.

#### Why?

Because it’s not an arbitrary test design decision to include variable flow. The application is what makes the choice to vary flow, which means the user would sometimes vary flow on the way through their one scenario. That means acceptance tests will also have to vary flow, as long as the flow traverses that decision point.

This test does traverse that decision point. So in this case, we’ve coded as if we have no idea if the optional page will pop up or not. For fully non-deterministic things or for things that have no “hint”, that’s a fallback.

But this is kind of artificial, since I’ve said that we *can’t* know if the flow needs to vary, it’s random. That doesn’t happen much outside of games and some types of event- or interrupt-driven code.

So some variations:

#### We don’t *care* why an extra page sometimes pops up.

In other words, it’s out of scope. Tests ultimately check a set of rules: given A, if B, then C. But not every test has to check every rule involved with what they’re testing, and usually shouldn’t. Otherwise, they can get too complicated and have poor isolation.

Instead, some you don’t test at all because they’re not really behavior you own (e.g., if I disable a button it turns gray, if you’re on a standard framework that handles that for you), or they’re being tested in some other test, or maybe because they’re low-level internal side effects and you’re strictly testing black box.

Sometimes they’re just prohibitively hard to determine from inside the test--it involves filesystem, but your test has no filesystem access, or involves a check of some other external that is hard to mock. Maybe it's easier to ignore it than accommodate it, even taking risk into account.

So the takeaway is sometimes you can decide you don’t care why. You just know it happens. You’re not testing the rule.

In this case, something like the above is OK. It’s not random, but you’ve decided the test doesn’t care why it happens so it may as well be.

You just make the test consistent with itself. You won’t be testing whatever rule causes `OPTIONAL_PAGE` to pop, but if it’s there you’ll verify it works correctly, and the test won’t error out based on the non-determinism.

Just don’t assume you know that `OPTIONAL_PAGE` pops correctly unless you test it elsewhere.

#### We do care, and the information is readily available to the test at runtime.

Ex: “If I have an active connection, according to the system, the optional page pops up.”

For a UI test, this usually means getting info from either from the back end or some UI you’re already on. You wouldn’t want to change the active dialog to fish out some information, since you’ve now blown the flow of your user scenario. But you might gather it at the beginning of the test then navigate to your starting point, or grab it from the status bar, or API, or whatever.

Since the information is gathered at runtime, this necessarily means parameterized logic, with the parameter being the available information. Splitting the test and choosing one or the other isn’t a possibility at all, because you won’t know what path you should have taken until you’re already inside the test.

So that middle block looks like:

    optional_condition = get_back_end_status(…)
    …
    if optional_condition:
        assert(vwizard.current_page == vwizard.OPTIONAL_PAGE, …)
        assert(…verify page…)
        vwizard.next()

What’s changed is that based on what I know now of the optional condition, I know whether to expect `OPTIONAL_PAGE`. I can verify based on that condition, that the system acted in a way that was self-consistent.

So now you’re testing a  different rule:

Given `optional_condition` is `True`, if I hit next, I land on `OPTIONAL_PAGE`.

What you’re not testing is the rule that leads to `optional_condition` being `True`.

But this can be OK, if the rule behind `optional_condition` isn’t something you’ve decided to care about. There’s a major advantage that your test will be much more robust for the things you do care about.

You can balance these concerns. Usually, internal consistency issues ultimately have a root. For example, once you have a SIM, the back end API registers it, and everything else is consistent with the back end API.

That makes one effective compromise to make almost all of your tests aim for internal runtime consistency, except for the one test elsewhere that verifies that the root reflects the external condition, or at least spot-checks the black-box side effects separately.

This keeps the majority of your tests very robust while still warning you if something is amiss. The individual test is less sensitive to that condition, but that's a good thing. Do you really want your whole suite failing if it doesn't have to, or just enough to warn you of the issue?

#### We do care, and we want or need to supply the information, but it will vary.

Ex: “If I’ve put a SIM in the phone, the optional page pops up. Some phones don’t have SIMs”

If there’s no way to gather the same info from elsewhere (let’s assume the back end can’t tell you if you have a SIM) then you have no choice. You need to supply the information to the test somehow.

And sometimes you might want to do this even if the information is available at runtime. Maybe you don't trust the system, or don't know if the accuracy of the runtime information is verified elsewhere.

There are a couple of ways to supply information.

One way is to have completely separate tests that you run based on what you know, probably choosing using manifest variables or similar.

But in this particular case we have a lot of code that’s always the same and a small amount of code that’s different based on the condition. That means either we’ll do a lot of copy/paste coding between two slightly different tests, or we’ll need to segment our test into reusable parts.

For this kind of test, segmenting isn’t an awful option. It’s a wizard, so you have a natural split between pages. So it’s conceivable to split out into `verify_page_start`, `verify_page_optional`, `verify_page_finish`, and have one test that wraps the tests for start and finish pages, and another that includes optional between them.

    def verify_page_start():
        assert(...verify page...)
        
    def verify_page_optional():
        # split out for symmetry
        assert(...verify page...)
        
    def verify_page_end():
        assert(...verify page...)
        
    def test_variable_wizard_with_sim():
        vwizard = VWizard.launch()
        verify_page_start()
        vwizard.next()
        verify_page_optional()
        vwizard.next()
        verify_page_end()
        vwizard.finish()
        
    def test_variable_wizard_without_sim():
        vwizard = VWizard.launch()
        verify_page_start()
        vwizard.next()
        verify_page_end()
        vwizard.finish()

That's a lot more code, but in a more realistic scenario where you do more per page, it won't be as much overhead. Maybe it's worth it.

However, not everything has a natural split like that. Some flows are simply linear within a single area, but have optional parts.

If there’s not a natural split, you’ll get `do_part_A`, `do_optional_part`, `do_part_B`, where the only thing defining them is the order of the test flow (steps 1-4 in A, steps 5-10 in B, some optional stuff in between).

This is often harder to understand than a mostly linear flow with a simple maybe block. You will need to think about how the functions flow from one to the other with and without optional bits. That’s true either way, but it’s harder when they’re split out and the code isn't adjacent.

I would only recommend splitting tests out when they have natural, cohesive splits, with any UI state changes being rather self-contained in cohesive areas.

Another way is to have a single test that has a maybe block, using a hint you explicitly send in:

    if testvars.has_a_sim:
        assert(vwizard.current_page == vwizard.OPTIONAL_PAGE, …)
        assert(…verify page…)
        vwizard.next()

In this case, we’re pulling from a test variables file. Depending on how your tests are run, you can also supply this as a parameter to the test function.

In either case, you’re now testing a new rule: Given *I’ve been told* I have a SIM, if I hit next, I land on `OPTIONAL_PAGE`.

The test is now checking that it’s consistent *with your knowledge* of the state of the system under test.


#### We do care, and want or need to supply the information, but we're sure it won’t vary.

Ex: “If I’ve put a SIM in the phone, the optional page pops up. All phones we will ever test have SIMs.”

This one’s easy, bake it into the test. This is just echoing the assumptions surrounding your usual, basic test code, after all.  At this point, you have no maybe block. You just assume a SIM is there, and `OPTIONAL_PAGE` isn’t really optional anymore.

    def test_not_variable_wizard():
        nvwizard = NVWizard.launch()
        assert(…verify page…)
        nvwizard.next()
        assert(…verify page…)
        nvwizard.next()
        assert(…verify page…)
        nvwizard.finish()

You’re still kind of testing a rule: Given *the test designer was right that I always have a SIM*, if I hit next, I land on `OPTIONAL_PAGE`.

The problem here is that this rule is now completely implied. The test no longer describes the general behavior of the system, only the behavior *plus your assumptions*. If someone doesn’t have a SIM, they’re going to be mightily confused when the test fails, so you’d better be right.

You're usually not.

So that brings us to:

#### We do care, and want or need to supply the information, but it *might* vary.

Ex: “If I’ve put a SIM in the phone, the optional page pops up. I think all phones we will ever test have SIMs, but I could be wrong”

Every time you think you have something that could technically vary but surely won't, it's probably really this. That's many times over true if it comes down to human decision-making or error.

The smart thing to do is account for that. This is what constants are made for.

So in this case, you take a hybrid approach between the last two methods.

    has_a_sim = True
    …
    if has_a_sim:
        assert(vwizard.current_page == vwizard.OPTIONAL_PAGE, …)
        assert(…verify page…)
        vwizard.next()

At first glance, this seems superfluous. If `has_a_sim` is assumed, why define a constant and put an if on it?

Because it solves the problems above:

* You’ve correctly documented the behavior of the system, independently of any assumptions you’ve put on its external state. The assumptions are encapsulated in `has_a_sim`.

* You’ve made it clear why, should your assumption be incorrect in the future, the test is now failing in that spot.

* You’ve given a single, obvious point of change for that assumption, especially necessary if you’re reusing it.

This has value even in simpler cases. Let’s imagine we’re verifying a status bar that shows an icon when a SIM is included.

    assert(status_bar.has_sim_icon, …)

vs.

    has_a_sim = True
    …
    assert(status_bar.has_sim_icon == has_a_sim, ...)

One of these is more accurate to the true behavior of the system than the other.

And later, if you decide to make this a variable via a defaults file, the change to the second one will be obvious and, importantly, something you can do a file search for. Searching on True or nothing is much harder.

## Some More Rules of Thumb:

### With rare exception, acceptance test code is written at a functional level of abstraction.

Tests should read more or less like you’d talk to a person. They usually shouldn’t be concerned with individual keystrokes, the details of menu navigation, and other strictly mechanical operations. These should be abstracted into app objects.

Similarly, tests should almost never be concerned with literal data. The test doesn’t care that “Ctrl-Shift-Meta-C” launches the calculator, it cares that “The menu sequence we’ve said belongs to the calculator launches it.” This sort of data should be abstracted elsewhere as well. At the very least, such information should be put in named constants.

This is the same logic as to why selectors don’t usually belong in tests either. Aside from complexity, they’re string literals and implementation details.

This is also usually true of raw lambda-based Waits; where they get exposed to the test, they should be wrapped with meaningful names:

    login.wait_for_Submit_is_not_allowed()
    
-or-

    login.wait_for_Submit_is_allowed(False)

-not-

    Wait(...).until(lambda m: return not login.submit_button.is_enabled())

If it’s a common wait, put it in the app object. If it’s very unusual and a one-off, it can live as a function in the test file, but either way it should be named with the intent in terms of the operation.

If you can’t read the test out loud like a standard test scenario, that’s a warning sign. Most well-written tests consist of a string of encapsulated function calls. Their value-add is ordering those calls and making verifications.

The exception will be tests specifically checking that a control matches a literal, or microchecking the behavior of the UI (e.g. when I tab from here, I end up there; or when I select this dropdown, this button becomes disabled).

Those types of tests end up talking directly to controls, creating more customized waits, etc. Keep that code out of your basic acceptance tests and put it in separate low-level tests.

### Tests are still code, and common code best practices still apply.

* Don’t use magic values unless you really mean to test against a raw value with no other semantic meaning. This is rare. Name your literal values if at all reasonable, even to use them once.

    The big exception here is error messages on assertions.

* Functions do one thing, especially in app objects, or else they’re not really functions. The one thing a test function does is run a single test scenario.

* Logic should be kept very simple in a given function. Long or winding functions are warning signs.

* Don’t construct strings via concatenation or substitution, especially ones referring to externals, like URLs, selectors, control names, etc.

    It adds an unnecessary level of complexity, and since you don’t actually own those strings they may change out from under you in a way that makes your construction not work anymore. Acceptance tests also often need to be localized so they can be run on l10n builds, and constructed strings aren’t localizable.

* Don’t rely on comments or error messages over proper naming and explicit, clear code.

    The code will be better reviewed and better maintained than comments or strings. Most people know to change the code if the intention has changed. They don’t necessarily change the comments or even error messages, assuming they even supply them.
    
    More to the point, if you have to read the code, realize you don't know what it means, read the error/comment, then read the code again, you've already read three times as much as you should have. Even worse, maybe you *think* you know what it means...but you're wrong. Fun times ahead.
    
    Finally, part of code review is checking that comments and error messages are accurate. If you can't understand the code without them, you won't be able to do this.
    
    Test code has no tests. Clarity and review is all we have. Optimize for it.

### And there are some test best practices as well:

* As above, flow logic is a warning sign in tests. Make sure it’s really the right thing to do. Most tests don't require logic.

    An exception is for loops in tests that generate something against the system. Those can be valuable for seeing if generating a lot of something causes issues.

* Generally, don’t verify against an expression. Assign expressions to variables, then verify that.

    This does not apply to simple unary operations like negating flags or simple comparisons like X == something, but does to most other kinds of expressions.

    The reasons to do this are avoiding yet another type of magic value (expressions are this as well if not very obvious) and because you can easily do a log or debug watch against a named value, but not against an inline expression.

* Only compute verification values if you’re testing a rule that is that computation

    If your rule is that when I take two pictures, I get two thumbnails but four photo files (HDR and non-HDR?) it’s perfectly appropriate to have:

        def test_take_photos(count):
            expected_thumbnails = count
            expected_files = 2 * count
            
            camera = Camera.launch()
            for i in xrange(count):
                camera.take_photo()
            assert(camera.get_thumbnail_count() == expected_thumbnails, …)
            assert(camera.get_file_count() == expected_files, …)

    Just keep the expression out of the assertion.
    
    This corresponds exactly to the rules:
    
    * Given _count_ times to do it, when I take that many photos I have that many thumbnails.
    * Given _count_ times to do it, when I take that many photos I have twice that many files

    You could choose to take `count` and `file_count` separately as parameters, but now you’re no longer testing the business rule, you’re testing that whoever supplied the information did it correctly according to the business rule.

### Make the tests explicit about their assumptions and their intent.

The last example above is explicit to a fault. I could have written:

    assert(camera.get_thumbnail_count() == count, …)
    assert(camera.get_file_count() == count * 2, …)

And that would still have been a correct test (aside from quibbles about the second assertion on a complex expression).

However, there's value in laying it out like I did in the last section. When reading the test, that one says:

1. I will expect as many thumbnails as times I take photos.
2. I will expect twice as many files as times I take photos
3. I'm taking photos this many times over.
4. I'm now checking that the real thumbnail count matches my expectations.
5. I'm now checking that the real file count matches my expectations.

This is more obvious than the shorter version:

1. I'm taking photos this many times over.
2. I'm now checking that the real thumbnail count matches how many times I took photos.
3. I'm now checking that the real file count matches twice as many times I took photos.

It also explains important relationships at the beginning, where they can help inform why the test was written that way to begin with. 

That makes this sort of pattern useful even in languages where normally you'd declare and assign variables closest to first use. Instead, you're declaring and assigning them closest to *first relevance to the reader*, the beginning of the test. 

This is a simple example, and it's frankly overkill for something this trivial. In real code, I might very well go with the second way.

But what if I didn't have a `count` variable?

    def test_take_photos():
        camera = Camera.launch()
        camera.take_a_photo()
        camera.take_a_photo()
        
        assert(camera.get_thumbnail_count() == 2, ...)
        assert(camera.get_file_count() == 4, ...)

This is about as simple as it gets. It's certainly concise and easy to read. I don't like the magic numbers, but we could fix that pretty easily with:

        expected_thumbnails = 2
        expected_files = 4

But even with that, now I have no real clue what the business rules are that are being verified. I'm smart enough to realize that probably means that the thumbnail count matches the number of photos, but the files... Do I expect photos * 2 = 4 or photos ^ 2 = 4 or photos + 2 = 4 or what? 

I wouldn't be able to review this for correctness--especially since 9 times out of 10, the error message will be something about expecting 4 files, not twice as many files as photos.

So let's take it one step further:

    def test_take_photos():
        photo_count = 2
        expected_thumbnails = photo_count
        expected_files = photo_count * 2
        
        camera = Camera.launch()
        for i in xrange(photo_count):
            camera.take_a_photo()
        
        assert(camera.get_thumbnail_count() == expected_thumbnails, ...)
        assert(camera.get_file_count() == expected_files, ...)

That's much clearer. The rules are enumerated at the top, the relationship between files and photos is clearly articulated, and if we ever decide the test should take a different number of photos or to parameterize it--common situations, both--it's an easy change.

We could remove `photo_count` and have:

    expected_thumbnails = 2
    expected_files = expected_thumbnails * 2
    
That would address my core issue with not documenting the relationship. But it's not quite accurate. Thumbnails correlate with files but they don't cause them. By introducing `photo_count` it's absolutely clear that these two things both depend on the same variable, but not each other. The test is accurately communicated.

There is a lot of value in splitting out the rules you are testing and getting them out of the assertions, even when they aren't complex expressions. You should never have to work backwards from an assertion to figure out the rule being tested or understand the test.

Remember that the goal is not just to understand the automation, it's to understand the intent of the automation and the assumptions designed into it. Always write to the intent and clearly communicate assumptions.
