=============
durable_rules
=============  
durable_rules is a polyglot micro-framework for real-time, consistent and scalable coordination of events. With durable_rules you can track and analyze information about things that happen (events) by combining data from multiple sources to infer more complicated circumstances.

A full forward chaining implementation (A.K.A. Rete) is used to evaluate facts and massive streams of events in real time. A simple, yet powerful meta-liguistic abstraction lets you define simple and complex rulesets as well as control flow structures such as flowcharts, statecharts, nested statecharts and time driven flows. 

The durable_rules core engine is implemented in C, which enables ultra fast rule evaluation as well as muti-language support. durable_rules relies on state of the art technologies: Werkzeug(http://werkzeug.pocoo.org/ is used to host rulesets). Inference state is cached using Redis(http://www.redis.io), This allows for fault tolerant execution and scale-out without giving up performance.

**Example 1**

durable_rules is simple: to define a rule, all you need to do is describe the event or fact pattern to match (antecedent) and the action to take (consequent).

In this example the rule can be triggered by posting `{"id": 1, "subject": "World"}` to url `http://localhost:5000/test/1`

::

    from durable.lang import *

    with ruleset('test'):
        # antecedent
        @when_all(m.subject == 'World')
        def say_hello(c):
            # consequent
            print ('Hello {0}'.format(c.m.subject))

    run_all()

**Example 2**

durable_rules provides string pattern matching. Expressions are compiled down to a DFA, guaranteeing linear execution time in the order of single digit nano seconds per character (note: backtracking expressions are not supported).

::

    from durable.lang import *

    with ruleset('test'):
        @when_all(m.subject.matches('3[47][0-9]{13}'))
        def amex(c):
            print ('Amex detected {0}'.format(c.m.subject))

        @when_all(m.subject.matches('4[0-9]{12}([0-9]{3})?'))
        def visa(c):
            print ('Visa detected {0}'.format(c.m.subject))

        @when_all(m.subject.matches('(5[1-5][0-9]{2}|222[1-9]|22[3-9][0-9]|2[3-6][0-9]{2}|2720)[0-9]{12}'))
        def mastercard(c):
            print ('Mastercard detected {0}'.format(c.m.subject))

    run_all()

**Example 3**

durable_rules super-power is the ability to define forward reasoning rules. In other words, rules to derive an action based on a set of correlated facts or observed events. The example below illustrates this basic building block by calculating the first 100 numbers of the Fibonacci series.

::

    from durable.lang import *

    with ruleset('fibonacci'):
        @when_all(c.first << (m.value != 0),
                  c.second << (m.id == c.first.id + 1))
        def calculate(c):
            print('Value: {0}'.format(c.first.value))
            if c.second.id > 100:
                print('Value: {0}'.format(c.second.value))
            else:
                c.assert_fact({'id': c.second.id + 1, 'value': c.first.value + c.second.value})
                c.retract_fact(c.first)
    
        @when_start
        def start(host):
            host.assert_fact('fibonacci', {'id': 1, 'sid': 1, 'value': 1})
            host.assert_fact('fibonacci', {'id': 2, 'sid': 1, 'value': 1})
        
    run_all()

**Example 4**

durable_rules can also be used to solve traditional Production Bussiness Rules problems. The example below is the 'Miss Manners' benchmark. Miss Manners has decided to throw a party. She wants to seat her guests such that adjacent guests are of opposite sex and share at least one hobby. 

Note how the benchmark flow structure is defined using a statechart to improve code readability without sacrificing performance nor altering the combinatorics required by the benchmark. For 128 guests, 438 facts, the execution time is less than 2 seconds. More details documented in this blog post (http://jruizblog.com/2015/07/20/miss-manners-and-waltzdb/).

::

    from durable.lang import *

    with statechart('miss_manners'):
        with state('start'):
            @to('assign')
            @when_all(m.t == 'guest')
            def assign_first_seating(c):
                c.s.count = 0
                c.s.g_count = 1000
                c.assert_fact({'t': 'seating',
                               'id': c.s.g_count,
                               's_id': c.s.count, 
                               'p_id': 0, 
                               'path': True, 
                               'left_seat': 1, 
                               'left_guest_name': c.m.name,
                               'right_seat': 1,
                               'right_guest_name': c.m.name})
                c.assert_fact({'t': 'path',
                               'id': c.s.g_count + 1,
                               'p_id': c.s.count, 
                               'seat': 1, 
                               'guest_name': c.m.name})
                c.s.count += 1
                c.s.g_count += 2
                print('assign {0}'.format(c.m.name))

        with state('assign'):
            @to('make')
            @when_all(c.seating << (m.t == 'seating') & 
                                   (m.path == True),
                      c.right_guest << (m.t == 'guest') & 
                                       (m.name == c.seating.right_guest_name),
                      c.left_guest << (m.t == 'guest') & 
                                      (m.sex != c.right_guest.sex) & 
                                      (m.hobby == c.right_guest.hobby),
                      none((m.t == 'path') & 
                           (m.p_id == c.seating.s_id) & 
                           (m.guest_name == c.left_guest.name)),
                      none((m.t == 'chosen') & 
                           (m.c_id == c.seating.s_id) & 
                           (m.guest_name == c.left_guest.name) & 
                           (m.hobby == c.right_guest.hobby)))
            def find_seating(c):
                c.assert_fact({'t': 'seating',
                               'id': c.s.g_count,
                               's_id': c.s.count, 
                               'p_id': c.seating.s_id, 
                               'path': False, 
                               'left_seat': c.seating.right_seat, 
                               'left_guest_name': c.seating.right_guest_name,
                               'right_seat': c.seating.right_seat + 1,
                               'right_guest_name': c.left_guest.name})
                c.assert_fact({'t': 'path',
                               'id': c.s.g_count + 1,
                               'p_id': c.s.count, 
                               'seat': c.seating.right_seat + 1, 
                               'guest_name': c.left_guest.name})
                c.assert_fact({'t': 'chosen',
                               'id': c.s.g_count + 2,
                               'c_id': c.seating.s_id,
                               'guest_name': c.left_guest.name,
                               'hobby': c.right_guest.hobby})
                c.s.count += 1
                c.s.g_count += 3

        with state('make'):
            @to('make')
            @when_all(cap(1000),
                      c.seating << (m.t == 'seating') & 
                                   (m.path == False),
                      c.path << (m.t == 'path') & 
                                (m.p_id == c.seating.p_id),
                      none((m.t == 'path') & 
                           (m.p_id == c.seating.s_id) & 
                           (m.guest_name == c.path.guest_name)))
            def make_path(c):
                for frame in c.m:
                    c.assert_fact({'t': 'path',
                                   'id': c.s.g_count,
                                   'p_id': frame.seating.s_id, 
                                   'seat': frame.path.seat, 
                                   'guest_name': frame.path.guest_name})
                    c.s.g_count += 1
                
            @to('check')
            @when_all(pri(1), (m.t == 'seating') & (m.path == False))
            def path_done(c):
                c.retract_fact(c.m)
                c.m.id = c.s.g_count
                c.m.path = True
                c.assert_fact(c.m)
                c.s.g_count += 1
                print('path sid: {0}, pid: {1}, left guest: {2}, right guest {3}'.format(c.m.s_id, c.m.p_id, c.m.left_guest_name, c.m.right_guest_name))

        with state('check'):
            @to('end')
            @when_all(c.last_seat << m.t == 'last_seat', 
                     (m.t == 'seating') & (m.right_seat == c.last_seat.seat))
            def done(c):
                print('end')
            
            to('assign')

        state('end')

**Reference Manual:**

- Python(https://github.com/jruizgit/rules/blob/master/docs/py/reference.md/)

**Blog:** 

- Miss Manners and Waltzdb (07/2015): http://jruizblog.com/2015/07/20/miss-manners-and-waltzdb/
- Polyglot (03/2015):http://jruizblog.com/2015/03/02/polyglot/
- Rete_D (02/2015):http://jruizblog.com/2015/02/23/rete_d/
- Boosting Performance with C (08/2014): http://jruizblog.com/2014/08/19/boosting-performance-with-c/
- Rete Meets Redis (02/2014):http://jruizblog.com/2014/02/02/rete-meets-redis/
- From Expert Systems to Cloud Scale Event Processing (01/2014):http://jruizblog.com/2014/01/27/event-processing/

