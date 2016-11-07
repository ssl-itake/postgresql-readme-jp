src/backend/utils/misc/README

GUC Implementation Notes
========================

GUC ������̒���
========================

The GUC (Grand Unified Configuration) module implements configuration
variables of multiple types (currently boolean, enum, int, real, and string).
Variable settings can come from various places, with a priority ordering
determining which setting is used.


GUC�iGrand Unified Configuration�j���W���[���́A
�����̃^�C�v�icurrently boolean, enum, int, real, and string�j�̐ݒ�ϐ����������Ă��܂��B
�ϐ��ݒ�́A�ݒ�����肷��D�揇�ʕt���ƁA�l�X�ȏꏊ���猈�肳�����̂ł��B

Per-Variable Hooks
------------------

�ϐ����̃t�b�N
------------------

Each variable known to GUC can optionally have a check_hook, an
assign_hook, and/or a show_hook to provide customized behavior.

GUC�ɔc������Ă���e�ϐ��́A�K�v�ɉ�����check_hook�Aassign_hook�������Ƃ��ł��A
�����/�܂���show_hook�́A�J�X�^�}�C�Y���ꂽ�����񋟂��܂��B


Check hooks are used to perform validity checking on variable values
(above and beyond what GUC can do), to compute derived settings when
nontrivial work is needed to do that, and optionally to "canonicalize"
user-supplied values.  

�`�F�b�N�t�b�N�́A�ݒ���s�����߂ɏd�v�ȍ�Ƃ��s�����߂ɕK�v�Ƃ����h���̐ݒ�̌v�Z�A
����сA�K�v�ɉ����Ẵ��[�U�w��l��"���K��"�����{���邽�߁A
�ϐ��̒l�Ó����iGUC�エ��т͒����ĉ����ł��邩�j�̃`�F�b�N�����s���邽�߂Ɏg�p����Ă��܂��B

Assign hooks are used to update any derived state
that needs to change when a GUC variable is set.  Show hooks are used to
modify the default SHOW display for a variable.

���蓖�ăt�b�N��GUC�ϐ����ݒ肳��Ă���Ƃ��ɁA
�ύX����K�v������C�ӂ̔h����Ԃ��X�V���邽�߂Ɏg�p����܂��B
�\���t�b�N�́A�ϐ��̃f�t�H���g��SHOW�̕\����ύX���邽�߂Ɏg�p����Ă��܂��B

If a check_hook is provided, it points to a function of the signature
	bool check_hook(datatype *newvalue, void **extra, GucSource source)
The "newvalue" argument is of type bool *, int *, double *, or char **
for bool, int/enum, real, or string variables respectively.  The check
function should validate the proposed new value, and return true if it is
OK or false if not.  The function can optionally do a few other things:

check_hook���񋟂���Ă���ꍇ�A����͉��L�̊֐��̒�`���|�C���g���Ă��܂��B
	bool check_hook(datatype *newvalue, void **extra, GucSource source)
"newvalue�����͂��ꂼ��bool *, int *, double *, �������� bool�^��char **, 
int/enum, real�������͂��ꂼ��̕�����ϐ��ł��B
�`�F�b�N�֐�����Ă��ꂽ�V�����l�����؂��A
���ꂪOK�ł���ꍇ��true�A�����łȂ��ꍇ��false�ŕԂ��K�v������܂��B
���̊֐��́A�K�v�ɉ����đ��̂������̂��Ƃ��s�����Ƃ��ł��܂��F

* When rejecting a bad proposed value, it may be useful to append some
additional information to the generic "invalid value for parameter FOO"
complaint that guc.c will emit.  To do that, call
	void GUC_check_errdetail(const char *format, ...)

* ������Ēl�����ۂ���ꍇ�́Aguc.c���o�͂����ʓI��"�p�����[�^FOO�ɖ����Ȓl"�̑i���ɁA
  �������̒ǉ�����t�^���邽�߂ɗL�p�ł���ꍇ������܂��B������s�����߂ɁA���L���Ăт܂��B
	void GUC_check_errdetail(const char *format, ...)

where the format string and additional arguments follow the rules for
errdetail() arguments.  The resulting string will be emitted as the
DETAIL line of guc.c's error report, so it should follow the message style
guidelines for DETAIL messages.  There is also
	void GUC_check_errhint(const char *format, ...)
which can be used in the same way to append a HINT message.
����������ƒǉ��̈����̉ӏ���errdetail�i�j�̈����̋K���ɏ]���Ă��������B
���ʕ������guc.c�̃G���[���|�[�g�̏ڍ׍s�Ƃ��ďo�͂���܂��̂ŁA
DETAIL���b�Z�[�W�̃��b�Z�[�W�`���K�C�h���C���ɏ]���Ă��������B
	void GUC_check_errhint(const char *format, ...)
�������@��HINT���b�Z�[�W��ǉ����邱�Ƃ��\�ł��B

Occasionally it may even be appropriate to override guc.c's generic primary
message or error code, which can be done with
	void GUC_check_errcode(int sqlerrcode)
	void GUC_check_errmsg(const char *format, ...)
guc.c�̈�ʓI�ȃv���C�}���[���b�Z�[�W��G���[�R�[�h���A
���L�̊֐��ŏ㏑�����邱�Ƃ��K�؂ł��邱�Ƃ�����܂��B
	void GUC_check_errcode(int sqlerrcode)
	void GUC_check_errmsg(const char *format, ...)

In general, check_hooks should avoid throwing errors directly if possible,
though this may be impractical to avoid for some corner cases such as
out-of-memory.
��ʓI�ɁAcheck_hooks�͉\�ł���΁A���ڃG���[�𓊂��Ȃ��悤�ɂ���K�v������܂��B
out-of-memory�̂悤�ȁA�ő��ɔ������Ȃ����ȃP�[�X�ł͉�����邱�Ƃ����������܂���B

* Since the newvalue is pass-by-reference, the function can modify it.
This might be used for example to canonicalize the spelling of a string
value, round off a buffer size to the nearest supported value, or replace
a special value such as "-1" with a computed default value.  
* newvalue�͎Q�Ɠn���ł���̂ŁA���̊֐��͂����ύX���邱�Ƃ��ł��܂��B
����́A���̂悤�ȓ��ʂȒl���A������l�̃X�y���𐳋K������A
�ł��߂��T�|�[�g�l�Ƀo�b�t�@�T�C�Y���l�̌ܓ�����A��������
���ʂȒl�ł���u-1�v���v�Z���ꂽ�f�t�H���g�l�ɒu�������邽�߂ɗ��p�����ł��傤�B

If the function wishes to replace a string value, it must malloc (not palloc)
the replacement value, and be sure to free() the previous value.
�֐��́A������l��ύX�������ꍇ�́A�u���l��malloc(palloc�ł͂Ȃ�)���A
�ȑO�̒l��free()�ŉ������悤�ɂ��ĉ������B

* Derived information, such as the role OID represented by a user name,
can be stored for use by the assign hook.  
* ���[�U���ɂ���ĕ\����郍�[��OID�̂悤�ȁA�����n���ꂽ���ɂ��Ă�
assing hook�ɂ��g�p�̂��߂ɕۑ����邱�Ƃ��ł��܂��B

To do this, malloc (not palloc)
storage space for the information, and return its address at *extra.
������s���ɂ́A���̊i�[�X�y�[�Xmalloc (palloc�łȂ�)���A
����*extra�̃A�h���X��ԋp���܂��B

guc.c will automatically free() this space when the associated GUC setting
is no longer of interest.  *extra is initialized to NULL before call, so
it can be ignored if not needed.
guc.c�́A�֘A����GUC�ݒ肪���͂�e�����Ȃ��ꍇ�ɁA
�����I�ɂ��̃X�y�[�X�����(free())���܂��B *extra�͌Ăяo���O��NULL�ɏ���������邽�߁A
�K�v�łȂ��ꍇ�́A����𖳎����邱�Ƃ��\�ł��B

The "source" argument indicates the source of the proposed new value,
If it is >= PGC_S_INTERACTIVE, then we are performing an interactive
assignment (e.g., a SET command). 
"�\�[�X"�����́A�V������Ēl�̃\�[�X�������Ă���A
���ꂪ >=PGC_S_INTERACTIVE�̏ꍇ�A��X�̓C���^���N�e�B�u�Ȋ��蓖�Ă����s���Ă��܂��B(�Ⴆ�΁ASET�R�}���h)

But when source < PGC_S_INTERACTIVE,
we are reading a non-interactive option source, such as postgresql.conf.
�������A�\�[�X��< PGC_S_INTERACTIVE�̏ꍇ�A
��X�́Apostgresql.conf�̂悤�ɁA��Θb�^�I�v�V�����\�[�X��ǂ�ł��܂��B

This is sometimes needed to determine whether a setting should be
allowed.  The check_hook might also look at the current actual value of
the variable to determine what is allowed.
����͎��܂ɐݒ�������邩�ǂ��������肷�邽�߂ɕK�v�Ƃ���܂��B
check_hook��������Ă��邩�𔻒f���邽�߂ɁA
���݂̕ϐ��̎��ۂ̒l���m�F���邱�Ƃ�����܂��B

Note that check hooks are sometimes called just to validate a value,
without any intention of actually changing the setting.  Therefore the
check hook must *not* take any action based on the assumption that an
assignment will occur.
���ۂ̐ݒ�ύX�̈Ӑ}���Ȃ��悤�ȏꍇ�ɂ��A�P�ɒl�̃o���f�[�g�̂��߁A
check hooks���Ă΂�邱�Ƃ����邱�ƂɋC��t���ĉ������B�]����
check hooks�́A���蓖�Ă���������Ƃ�������Ɋ�Â��ĔC�ӂ̓�������s���Ă�*�Ȃ�܂���*�B

If an assign_hook is provided, it points to a function of the signature
	void assign_hook(datatype newvalue, void *extra)
where the type of "newvalue" matches the kind of variable, and "extra"
is the derived-information pointer returned by the check_hook (always
NULL if there is no check_hook).  This function is called immediately
before actually setting the variable's value (so it can look at the actual
variable to determine the old value, for example to avoid doing work when
the value isn't really changing).

assign_hook���񋟂����ꍇ�A����͉��L�֐��̒�`���|�C���g���܂��B
	void assign_hook(datatype newvalue, void *extra)
�����ł́unewvalue�v�̌^�́A�ϐ��̎�ʂɈ�v���A�h�����|�C���^"extra"��
check_hook�ɂ���ĕԂ���܂�(check_hook���Ȃ��ꍇ�͏��NULL)�B���̊֐��́A
���ۂɂ͕ϐ��̒l��ݒ肷�钼�O�ɌĂяo����܂��B(�Â��l�����肷�邽�߂ɁA
���ۂ̕ϐ������邱�Ƃ��ł��܂��B�Ⴆ�Βl�����ۂɕύX����Ă��Ȃ��ꍇ�ɁA
��Ƃ��s�����Ƃ�����邱�Ƃ��ł��܂��B)

Note that there is no provision for a failure result code. assign_hooks
should never fail except under the most dire circumstances, since a failure
may for example result in GUC settings not being rolled back properly during
transaction abort.  
���s���ʃR�[�h�Ɋւ���K�肪���݂��Ȃ����Ƃɒ��ӂ��Ă��������B
assign_hooks�́A��Q���Ⴆ��GUC�ݒ�������炷�\�������邽�߁A
�g�����U�N�V�����A�{�[�g���ɓK�؂Ƀ��[���o�b�N����Ȃ����߁A
�ł��ߎS�ȏ󋵂������Ď��s���邱�Ƃ͂���܂���B

In general, try to do anything that could conceivably
fail in a check_hook instead, and pass along the results in an "extra"
struct, so that the assign hook has little to do beyond copying the data to
someplace.  
��ʓI�ɁAcheck_hook�ɂ����鎸�s�̑���ɍl���������̉������悤�ƁA
"extra"�\���̓��Ɍ��ʂ�`���܂��B���̂��߁Aassign hook�͂ǂ����ւ̃f�[�^�R�s�[�𒴂��āA
�s�����Ƃ͂قƂ�ǂ���܂���B

This applies particularly to catalog lookups: any required
lookups must be done in the check_hook, since the assign_hook may be
executed during transaction rollback when lookups will be unsafe.

����́A�J�^���O�̌����ɓ��ɓK�p����܂�:�J�^���O���������S�ł͂Ȃ����A
assign_hook�̓g�����U�N�V�����̃��[���o�b�N���Ɏ��s���邱�Ƃ��ł���̂ŁA
�K�v�Ȍ����́Acheck_hook�ōs��Ȃ���΂Ȃ�܂���B

Note that check_hooks are sometimes called outside any transaction, too.
This happens when processing the wired-in "bootstrap" value, values coming
from the postmaster command line or environment, or values coming from
postgresql.conf.  Therefore, any catalog lookups done in a check_hook
should be guarded with an IsTransactionState() test, and there must be a
fallback path to allow derived values to be computed during the first
subsequent use of the GUC setting within a transaction.  A typical
arrangement is for the catalog values computed by the check_hook and
installed by the assign_hook to be used only for the remainder of the
transaction in which the new setting is made.  Each subsequent transaction
looks up the values afresh on first use.  This arrangement is useful to
prevent use of stale catalog values, independently of the problem of
needing to check GUC values outside a transaction.

check_hooks�����X�A�g�����U�N�V�����̊O���ŌĂ΂�邱�Ƃɒ��ӂ��Ă�������
�ubootstrap�v�̒l����������ꍇ�ɏ�L��肪�������Apostmaster�̃R�}���h���C���A
���ˑ��A�܂�postgresql.conf���痈���l����������ꍇ�ɔ������܂��B
�]���āAcheck_hook�Ŏ��{�����C�ӂ̃J�^���O�����́A
IsTransactionState�i�j�e�X�g�ŃK�[�h�����ׂ��ł���A
����ꂽ�l�́A�g�����U�N�V��������GUC�ݒ�̍ŏ��̌㑱�̗��p���ɁA
�v�Z�ł���悤�ɂ��邽�߂̃t�H�[���o�b�N�p�X�����݂��Ȃ���΂Ȃ�܂���B
�T�^�I�ȍ\����check_hook�ɂ���Čv�Z���ꂽ�J�^���O�l�̂��߂̂��̂ł���A
assign_hook�ɂ��C���X�g�[���͐V�����ݒ肪�Ȃ���Ă���A
�g�����U�N�V�����̎c��̕����ɂ̂ݎg�p����܂��B�㑱�̊e�g�����U�N�V�����́A
�ŏ��̎g�p���ɐV���ɒl���������܂��B���̍\���́A�Â��Ȃ����J�^���O�l�̎g�p��h�~�����
�g�����U�N�V�����O��GUC�l�̃`�F�b�N���K�v�ł�����Ƃ͊֌W�Ȃ��L�p�ł��B

If a show_hook is provided, it points to a function of the signature
	const char *show_hook(void)
This hook allows variable-specific computation of the value displayed
by SHOW (and other SQL features for showing GUC variable values).
The return value can point to a static buffer, since show functions are
not used re-entrantly.

show_hook���񋟂���Ă���ꍇ�A����͉��L�̊֐��̒�`���|�C���g���Ă��܂�
	const char *show_hook(void)
���̃t�b�N�́ASHOW�i�����GUC�ϐ��̒l��\�����邽�߂́A���̑���SQL�@�\�j
�ŕ\�������l�̉όŗL�̌v�Z���\�ɂ��܂��B
show�֐��́A���G���g�����g�Ɏg�p����Ȃ��̂ŁA
�߂�l�́A�ÓI�o�b�t�@���w�����Ƃ��ł��܂��B

Saving/Restoring GUC Variable Values
------------------------------------

Prior values of configuration variables must be remembered in order to deal
with several special cases: RESET (a/k/a SET TO DEFAULT), rollback of SET
on transaction abort, rollback of SET LOCAL at transaction end (either
commit or abort), and save/restore around a function that has a SET option.
RESET is defined as selecting the value that would be effective had there
never been any SET commands in the current session.

�Z�[�u/���X�g�A GUC�ϐ��̒l
------------------------------------

�ݒ�ϐ��̑O�̒l�́A�������̓��ʂȏꍇ�ɑΏ����邽�߂�
�Y��Ă͂Ȃ�܂���: RESET (SET TO DEFAULT�Ƃ��Ēm����)�A 
SET�ɂ�����g�����U�N�V�����A�{�[�g�̃��[���o�b�N�A
�g�����U�N�V�����̍Ō�(commit��������abort)�ɂ�����A
SET LOCAL�̃��[���o�b�N�A������SET�I�v�V�����������Ă���Z�[�u/���X�g�A���ӂ̋@�\�B
RESET�́A���݂̃Z�b�V�������̔C�ӂ�SET�R�}���h�����s����Ȃ������ꍇ�ɁA
���ʓI�ł��낤�l��I������悤�ɒ�`����Ă��܂��B

To handle these cases we must keep track of many distinct values for each
variable.  The primary values are:
���̂悤�ȏꍇ���������߂Ɏ������́A���ꂼ��̕ϐ��̂��߂ɑ����̈قȂ�l��ǐՂ���K�v������܂��B
��v�Ȓl�͎��̂Ƃ���ł�:

* actual variable contents	always the current effective value

* reset_val			the value to use for RESET

* actual variable contents	���݂̗L���l

* reset_val			�l��RESET�ɂĎg�p

(Each GUC entry also has a boot_val which is the wired-in default value.
This is assigned to the reset_val and the actual variable during
InitializeGUCOptions().  The boot_val is also consulted to restore the
correct reset_val if SIGHUP processing discovers that a variable formerly
specified in postgresql.conf is no longer set there.)

(�eGUC�G���g�������C���[�h�ł̃f�t�H���g�l�ł���boot_val�������Ă��܂��B
����́AInitializeGUCOptions()�ɂ�reset_val�Ǝ��ۂ̕ϐ��ɑ������Ă��܂��B
boot_val��SIGHUP�������ȑO��postgresql.conf���Ŏw�肳�ꂽ�ϐ����A
���͂�ݒ肳��Ă��邱�Ƃ����������ꍇ�ɁA������reset_val�𕜌����邽�߂ɂ��Q�Ƃ���܂��B)

In addition to the primary values, there is a stack of former effective
values that might need to be restored in future.  Stacking and unstacking
is controlled by the GUC "nest level", which is zero when outside any
transaction, one at top transaction level, and incremented for each
open subtransaction or function call with a SET option.  A stack entry
is made whenever a GUC variable is first modified at a given nesting level.
(Note: the reset_val need not be stacked because it is only changed by
non-transactional operations.)

�v���C�}���l�ɉ����āA�����I�Ƀ��X�g�A�̂��߂�
�K�v�ɂȂ�ꍇ������O�̗L���l�̃X�^�b�N�����݂��܂��B
�X�^�b�L���O����уA���X�^�b�L���O��GUC "nest level"�ɂ�萧�䂳��A
0�̏ꍇ�A���ׂẴg�����U�N�V�������O���A�g�b�v�g�����U�N�V�����E���x����1�A����
SET�I�v�V�������w�肵���I�[�v���T�u�g�����U�N�V�����܂��͊֐��Ăяo���ł��B
GUC�ϐ����ŏ��ɗ^����ꂽ�l�X�g�E���x���ŕύX�����x�ɃX�^�b�N�G���g�����쐬����܂��B
(����: reset_val����g�����U�N�V��������ɂ���Ă̂ݕύX����邽�߁A
reset_val���X�^�b�N����K�v�͂���܂���B)

A stack entry has a state, a prior value of the GUC variable, a remembered
source of that prior value, and depending on the state may also have a
"masked" value.  The masked value is needed when SET followed by SET LOCAL
occur at the same nest level: the SET's value is masked but must be
remembered to restore after transaction commit.

�X�^�b�N�G���g���͏�ԁAGUC�ϐ��̑O�̒l�A�O�̒l�̊o�����\�[�X��ێ����A
�����āA��Ԃɉ����āA�u�}�X�N���ꂽ�v�l��ێ����ł��܂��B�}�X�N���ꂽ�l�́A
SET LOCAL������SET�������l�X�g�E���x���Ŕ��������Ƃ��ɕK�v�Ƃ���܂�:
SET�l�̓}�X�N����Ă��܂����A�g�����U�N�V�����̃R�~�b�g��Ƀ��X�g�A���邱�Ƃ�Y��Ă͂Ȃ�܂���B

During initialization we set the actual value and reset_val based on
whichever non-interactive source has the highest priority.  They will
have the same value.
���������Ɏ������͎��ۂ̒l��reset_val�A�ł��D��x�̍�����Θb�\�[�X�Ɋ�Â��Đݒ肵�܂��B

The possible transactional operations on a GUC value are:
GUC�l�ɂ����ĉ\�ȃg�����U�N�V��������́A���̂Ƃ���ł�:

Entry to a function with a SET option:
SET�I�v�V���������֐��ւ̃G���g���F

	Push a stack entry with the prior variable value and state SAVE,
	then set the variable.
	�O�̕ϐ��̒l�ƃZ�[�u��ԂƂ̃X�^�b�N�G���g�����v�b�V�����A
	���̕ϐ���ݒ肵�܂��B

Plain SET command:
�v���[��SET�R�}���h�F

	If no stack entry of current level:
		Push new stack entry w/prior value and state SET
	else if stack entry's state is SAVE, SET, or LOCAL:
		change stack state to SET, don't change saved value
		(here we are forgetting effects of prior set action)
	else (entry must have state SET+LOCAL):
		discard its masked value, change state to SET
		(here we are forgetting effects of prior SET and SET LOCAL)
	Now set new value.
	
	���݂̃��x���̃X�^�b�N�G���g�����Ȃ��ꍇ�F
		�V�����X�^�b�N�G���g���ɑO�̒l��SET��Ԃ��v�b�V������
	�X�^�b�N�G���g���̏�Ԃ�SAVE�ASET�A�܂���LOCAL�ł���ꍇ�F
		SET���邽�߃X�^�b�N�̏�Ԃ�ύX�A�ۑ����ꂽ�l��ύX���Ȃ�
	�����łȂ��ꍇ�i�G���g���́ASET+ LOCAL��Ԃ����K�v������j:
		���̃}�X�N���ꂽ�l��j�����A��Ԃ�SET�ɕύX���܂�
	�����ĐV�����l��ݒ肵�܂��B	
	
SET LOCAL command:
SET LOCAL�R�}���h;

	If no stack entry of current level:
		Push new stack entry w/prior value and state LOCAL
	else if stack entry's state is SAVE or LOCAL or SET+LOCAL:
		no change to stack entry
		(in SAVE case, SET LOCAL will be forgotten at func exit)
	else (entry must have state SET):
		put current active into its masked slot, set state SET+LOCAL
	Now set new value.
	
	���݂̃��x���̃X�^�b�N�G���g�����Ȃ��ꍇ�F
		�V�����X�^�b�N�G���g���ɑO�̒l��SET��Ԃ��v�b�V������
	�X�^�b�N�G���g���̏�Ԃ�SAVE�ALOCAL�A�܂���SET+LOCAL�ł���ꍇ�F
		�X�^�b�N�G���g����ύX���Ȃ�
		(SAVE�̏ꍇ�ɂ����ẮASET LOCAL�͊֐��I�����ɖY�����)
	�����łȂ��ꍇ�i�G���g���́ASET��Ԃ����K�v������j:
		�}�X�N���ꂽ�X���b�g�ɁA���݂̃A�N�e�B�u�����A��Ԃ�SET+LOCAL�ɐݒ肵�܂�
	�����ĐV�����l��ݒ肵�܂��B
	

Transaction or subtransaction abort:
�g�����U�N�V�����܂��̓T�u�g�����U�N�V�������A�{�[�g:

	Pop stack entries, restoring prior value, until top < subxact depth
	�X�^�b�N�G���g�����|�b�v���Atop < subxact depth�܂őO�̒l�����X�g�A���܂�

Transaction or subtransaction commit (incl. successful function exit):
�g�����U�N�V�����܂��̓T�u�g�����U�N�V�������R�~�b�g (�֐��̐���I�����܂�)

	While stack entry level >= subxact depth
		if entry's state is SAVE:
			pop, restoring prior value
		else if level is 1 and entry's state is SET+LOCAL:
			pop, restoring *masked* value
		else if level is 1 and entry's state is SET:
			pop, discarding old value
		else if level is 1 and entry's state is LOCAL:
			pop, restoring prior value
		else if there is no entry of exactly level N-1:
			decrement entry's level, no other state changeprior
		else
			merge entries of level N-1 and N as specified below
	
	stack entry level >= subxact depth�̊�
		�G���g���̏�Ԃ�SAVE�̏ꍇ�F
			�|�b�v���A�O�̒l�����X�g�A���܂�
		���x��1���G���g����Ԃ�SET+LOCAL�̏ꍇ:
			�|�b�v���A*�}�X�N���ꂽ*�l�����X�g���܂�
		���x��1���G���g����Ԃ�SET�̏ꍇ:
			�|�b�v���A�Â��l��p�����܂�
		���x��1���G���g����Ԃ�LOCAL�̏ꍇ:
			�|�b�v���A�O�̒l�����X�g�A���܂�
		�G���g�����Ȃ��A���x����N-1�̏ꍇ
			�G���g�����x�����f�N�������g���A���̏�ԕω��͂���܂���
		���̑��̏ꍇ
			�G���g�����x����N-1,N,N���}�[�W���A�ȉ��Ɏw�肳��܂��B

The merged entry will have level N-1 and prior = older prior, so easiest
to keep older entry and free newer.  There are 12 possibilities since
we already handled level N state = SAVE:
�}�[�W���ꂽ�G���g���̓��x�� N-1���O�̒l = �Â��O�̒l�������Ă��܂��B���̂��߁A
�Â��G���g����ێ����A�V�����l��������邱�Ƃ��ł��ȒP�ł��B
��X�͊���level N state = SAVE�̏�Ԉ������߁A12�̉\��������܂��B

N-1		N

SAVE		SET		discard top prior, set state SET
SAVE		LOCAL		discard top prior, no change to stack entry
SAVE		SET+LOCAL	discard top prior, copy masked, state S+L

SET		SET		discard top prior, no change to stack entry
SET		LOCAL		copy top prior to masked, state S+L
SET		SET+LOCAL	discard top prior, copy masked, state S+L

LOCAL		SET		discard top prior, set state SET
LOCAL		LOCAL		discard top prior, no change to stack entry
LOCAL		SET+LOCAL	discard top prior, copy masked, state S+L

SET+LOCAL	SET		discard top prior and second masked, state SET
SET+LOCAL	LOCAL		discard top prior, no change to stack entry
SET+LOCAL	SET+LOCAL	discard top prior, copy masked, state S+L


N-1		N

SAVE		SET		top prior��j�����A��Ԃ�SET�ɐݒ肷��
SAVE		LOCAL		top prior��j�����A�X�^�b�N�G���g���ւ̕ύX���s��Ȃ�
SAVE		SET+LOCAL	top prior��j����,�R�s�[�̓}�X�N����A��Ԃ�S+L

SET		SET		top prior��j�����A�X�^�b�N�G���g���ւ̕ύX���s��Ȃ�
SET		LOCAL		top prior��j�����A�X�^�b�N�G���g���ւ̕ύX���s��Ȃ�
SET		SET+LOCAL	top prior��j����,�R�s�[�̓}�X�N����A��Ԃ�S+L

LOCAL		SET		top prior��j�����A��Ԃ�SET�ɐݒ肷��
LOCAL		LOCAL		top prior��j�����A�X�^�b�N�G���g���ւ̕ύX���s��Ȃ�
LOCAL		SET+LOCAL	top prior��j����,�R�s�[�̓}�X�N����A��Ԃ�S+L

SET+LOCAL	SET		discard top prior and second masked, state SET
SET+LOCAL	LOCAL		top prior��j�����A�X�^�b�N�G���g���ւ̕ύX���s��Ȃ�
SET+LOCAL	SET+LOCAL	top prior��j����,�R�s�[�̓}�X�N����A��Ԃ�S+L

RESET is executed like a SET, but using the reset_val as the desired new
value.  (We do not provide a RESET LOCAL command, but SET LOCAL TO DEFAULT
has the same behavior that RESET LOCAL would.)  The source associated with
the reset_val also becomes associated with the actual value.

If SIGHUP is received, the GUC code rereads the postgresql.conf
configuration file (this does not happen in the signal handler, but at
next return to main loop; note that it can be executed while within a
transaction).  New values from postgresql.conf are assigned to actual
variable, reset_val, and stacked actual values, but only if each of
these has a current source priority <= PGC_S_FILE.  (It is thus possible
for reset_val to track the config-file setting even if there is
currently a different interactive value of the actual variable.)

The check_hook, assign_hook and show_hook routines work only with the
actual variable, and are not directly aware of the additional values
maintained by GUC.


GUC Memory Handling
-------------------

String variable values are allocated with malloc/strdup, not with the
palloc/pstrdup mechanisms.  We would need to keep them in a permanent
context anyway, and malloc gives us more control over handling
out-of-memory failures.

We allow a string variable's actual value, reset_val, boot_val, and stacked
values to point at the same storage.  This makes it slightly harder to free
space (we must test whether a value to be freed isn't equal to any of the
other pointers in the GUC entry or associated stack items).  The main
advantage is that we never need to malloc during transaction commit/abort,
so cannot cause an out-of-memory failure there.

"Extra" structs returned by check_hook routines are managed in the same
way as string values.  Note that we support "extra" structs for all types
of GUC variables, although they are mainly useful with strings.


GUC and Null String Variables
-----------------------------

A GUC string variable can have a boot_val of NULL.  guc.c handles this
unsurprisingly, assigning the NULL to the underlying C variable.  Any code
using such a variable, as well as any hook functions for it, must then be
prepared to deal with a NULL value.

However, it is not possible to assign a NULL value to a GUC string
variable in any other way: values coming from SET, postgresql.conf, etc,
might be empty strings, but they'll never be NULL.  And SHOW displays
a NULL the same as an empty string.  It is therefore not appropriate to
treat a NULL value as a distinct user-visible setting.  A typical use
for a NULL boot_val is to denote that a value hasn't yet been set for
a variable that will receive a real value later in startup.

If it's undesirable for code using the underlying C variable to have to
worry about NULL values ever, the variable can be given a non-null static
initializer as well as a non-null boot_val.  guc.c will overwrite the
static initializer pointer with a copy of the boot_val during
InitializeGUCOptions, but the variable will never contain a NULL.
