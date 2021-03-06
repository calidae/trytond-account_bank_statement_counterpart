================================
Account Bank Statement  Scenario
================================

Imports::

    >>> import datetime
    >>> import pytz
    >>> from dateutil.relativedelta import relativedelta
    >>> from decimal import Decimal
    >>> from operator import attrgetter
    >>> from proteus import Model, Wizard
    >>> from trytond.tests.tools import activate_modules
    >>> from trytond.modules.company.tests.tools import create_company, \
    ...     get_company
    >>> from trytond.modules.account.tests.tools import create_fiscalyear, \
    ...     create_chart, get_accounts
    >>> today = datetime.date.today()
    >>> now = datetime.datetime.now()

Install account_bank_statement_counterpart Module::

    >>> config = activate_modules('account_bank_statement_counterpart')

Create company::

    >>> _ = create_company()
    >>> company = get_company()
    >>> company.timezone = 'Europe/Madrid'
    >>> company.save()

Create fiscal year::

    >>> fiscalyear = create_fiscalyear(company)
    >>> fiscalyear.click('create_period')
    >>> period = fiscalyear.periods[0]

Create chart of accounts::

    >>> _ = create_chart(company)
    >>> accounts = get_accounts(company)
    >>> receivable = accounts['receivable']
    >>> revenue = accounts['revenue']
    >>> expense = accounts['expense']
    >>> cash = accounts['cash']
    >>> cash.bank_reconcile = True
    >>> cash.save()

Create party::

    >>> Party = Model.get('party.party')
    >>> party = Party(name='Party')
    >>> party.save()

Create Journals::

    >>> Sequence = Model.get('ir.sequence')
    >>> sequence = Sequence(name='Bank', code='account.journal',
    ...     company=company)
    >>> sequence.save()
    >>> AccountJournal = Model.get('account.journal')
    >>> account_journal = AccountJournal(name='Statement',
    ...     type='cash',
    ...     sequence=sequence,
    ... )
    >>> account_journal.save()
    >>> StatementJournal = Model.get('account.bank.statement.journal')
    >>> statement_journal = StatementJournal(name='Test',
    ...     journal=account_journal, account=cash,
    ...     )
    >>> statement_journal.save()

Create Move::

    >>> period = fiscalyear.periods[0]
    >>> Move = Model.get('account.move')
    >>> move = Move()
    >>> move.period = period
    >>> move.journal = account_journal
    >>> move.date = period.start_date
    >>> line = move.lines.new()
    >>> line.account = receivable
    >>> line.debit = Decimal('80.0')
    >>> line.party = party
    >>> line2 = move.lines.new()
    >>> line2.account = revenue
    >>> line2.credit = Decimal('80.0')
    >>> move.click('post')
    >>> move.state
    'posted'

Create Bank Move::

    >>> reconcile1, = [l for l in move.lines if l.account == receivable]

Create Bank Statement::

    >>> BankStatement = Model.get('account.bank.statement')
    >>> statement = BankStatement(journal=statement_journal, date=now)

Create Bank Statement Lines::

    >>> statement_line = statement.lines.new()
    >>> statement_line.date = now
    >>> statement_line.description = 'Statement Line'
    >>> statement_line.amount = Decimal('80.0')
    >>> statement.save()
    >>> statement.reload()
    >>> statement.state
    'draft'
    >>> statement.click('confirm')
    >>> statement_line, = statement.lines
    >>> statement_line.state
    'confirmed'
    >>> statement_line.account_date_utc != statement_line.account_date
    True
    >>> timezone = pytz.timezone('Europe/Madrid')
    >>> date = timezone.localize(statement_line.account_date_utc)
    >>> account_date = statement_line.account_date_utc + date.utcoffset()
    >>> statement_line.account_date == account_date
    True
    >>> reconcile1.bank_statement_line_counterpart = statement_line
    >>> reconcile1.save()
    >>> reconcile1.reload()
    >>> statement_line.click('post')
    >>> statement_line.state
    'posted'
    >>> move_line, = [x for x in reconcile1.reconciliation.lines if x !=
    ...    reconcile1]
    >>> move_line.account == reconcile1.account
    True
    >>> move_line.credit
    Decimal('80.0')
    >>> move_line2, = [x for x in move_line.move.lines if x != move_line]
    >>> move_line2.account == statement_line.account
    True
    >>> move_line2.debit
    Decimal('80.0')
    >>> receivable.reload()
    >>> receivable.balance
    Decimal('0.00')

Cancel the line and theck all the moves have been cleared::

    >>> statement_line.click('cancel')
    >>> len(statement_line.counterpart_lines)
    0
    >>> len(statement_line.bank_lines)
    0
    >>> receivable.reload()
    >>> receivable.balance
    Decimal('80.00')
