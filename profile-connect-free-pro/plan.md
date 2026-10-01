# Profile Connect Limited Free Tier Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the basic Profile Connect assignment workflow into the Free plugin while keeping advanced external data mapping in Pro.

**Architecture:** Add a Free/core admin service for assigning existing WordPress users to User Registration forms by writing or deleting `ur_form_id`. Keep the advanced external field/table mapping settings and mapped-value migration workflows out of Free. Reuse the current Users table and user profile edit surfaces so the feature feels native to WordPress and to the existing URM admin UI.

**Tech Stack:** WordPress admin PHP, User Registration & Membership Free, current Profile Connect addon source, WP users table hooks, user meta, PHPUnit/WP test suite where available, Playwright/manual browser verification.

**Spec:** Live prototype: `https://bhattaganesh.github.io/ur-prototypes/profile-connect-free-pro/`

## Global Constraints

- This implements the limited Profile Connect in Free approach, not the default-form fallback approach.
- Free includes only user-to-form assignment and disconnection.
- Pro keeps external field mapping, external table mapping, and advanced migration/import workflows.
- The feature writes/deletes the existing `ur_form_id` user meta key.
- Existing `ur_form_id` behavior and consumers must remain backward compatible.
- Do not move dynamic external-table SQL into Free.
- Do not silently auto-assign users without an explicit admin action.
- Do not break sites where the separate `user-registration-profile-connect` addon is already installed.
- Do not leave user-facing copy that says basic existing-user adoption requires buying Profile Connect.

## Review Focus

- Users without `ur_form_id`: selected users should become connected to the chosen form after a valid bulk action.
- Users with an existing `ur_form_id`: selected users should be reassigned only after an explicit admin action.
- Invalid/deleted form IDs: the assignment action should reject the request and avoid writing bad meta.
- Unauthorized users: users without the required capability should not see or execute connect/disconnect actions.
- Separate addon active: hooks should not duplicate controls or process the same request twice.

---

## File Structure

**Free plugin files to modify or create**

- Create: `includes/admin/class-ur-admin-profile-connect.php`
  - Owns Free basic Profile Connect UI and processing.
  - Registers Users table dropdowns, profile edit field, notices, and request handlers.

- Modify: `includes/admin/class-ur-admin.php`
  - Loads `UR_Admin_Profile_Connect` in admin when the current user can manage users.

- Modify: `includes/admin/settings/class-ur-settings-registration-login.php`
  - Stops routing `profile-connect` to the generic Pro upsell.
  - Returns the Free Profile Connect guidance/settings content for the `profile-connect` section.

- Modify: `includes/admin/class-ur-admin-user-list-manager.php`
  - Rewrites or suppresses the non-URM users upsell notice once basic Profile Connect is available in Free.

- Modify: `templates/myaccount/form-edit-profile-non-urm-user.php`
  - Rewrites the admin-only copy that currently tells admins to buy the Profile Connect addon.

- Modify: `includes/functions-ur-core.php`
  - Removes Profile Connect from Free upsell metadata or changes the metadata to upsell only advanced mapping/import.

**Profile Connect addon files to update**

- Modify: `../user-registration-profile-connect/includes/class-user-registration-profile-connect.php`
  - Prevent duplicate basic connect hooks when the Free plugin already provides them.
  - Keep advanced mapping features available for Pro/addon users.

- Modify: `../user-registration-profile-connect/includes/class-user-registration-profile-connect-process.php`
  - Remove or gate basic connect UI/processing if Free provides it.
  - Keep advanced mapping processing only where needed.

- Modify: `../user-registration-profile-connect/includes/settings/class-user-registration-profile-connect-settings.php`
  - Keep external field/table mapping as the advanced Pro/addon surface.

**Tests to add or update**

- Create: `tests/php/admin/class-ur-admin-profile-connect-test.php`
  - Unit/integration tests for capabilities, nonce validation, valid form validation, bulk assignment, disconnection, and profile save behavior.

- Update existing admin/user tests if the repository already has a better matching test location.

---

## Task 1: Add The Free Basic Profile Connect Admin Service

**Files:**
- Create: `includes/admin/class-ur-admin-profile-connect.php`
- Modify: `includes/admin/class-ur-admin.php`
- Test: `tests/php/admin/class-ur-admin-profile-connect-test.php`

**Interfaces:**
- Consumes: `ur_get_all_user_registration_form()`, `get_post()`, `update_user_meta()`, `delete_user_meta()`, WordPress Users table hooks.
- Produces: `UR_Admin_Profile_Connect` class with `init()`, `render_bulk_connect_control()`, `process_bulk_connect_request()`, `render_profile_field()`, `save_profile_field()`, and `is_valid_registration_form_id()`.

- [ ] **Step 1: Write failing tests for form validation**

```php
public function test_valid_registration_form_id_rejects_missing_post() {
	$service = new UR_Admin_Profile_Connect();

	$this->assertFalse( $service->is_valid_registration_form_id( 999999 ) );
}

public function test_valid_registration_form_id_accepts_user_registration_post() {
	$form_id = self::factory()->post->create(
		array(
			'post_type'   => 'user_registration',
			'post_status' => 'publish',
		)
	);

	$service = new UR_Admin_Profile_Connect();

	$this->assertTrue( $service->is_valid_registration_form_id( $form_id ) );
}
```

- [ ] **Step 2: Run the targeted tests and verify they fail**

Run the repository's PHP test command for the new test file. If the repo does not have a configured test runner, run `php -l includes/admin/class-ur-admin-profile-connect.php` after creating the file and document the missing test runner.

Expected before implementation: class or method not found.

- [ ] **Step 3: Create the admin service skeleton**

```php
<?php
/**
 * Admin Profile Connect basic assignment.
 *
 * @package UserRegistration\Admin
 */

defined( 'ABSPATH' ) || exit;

class UR_Admin_Profile_Connect {

	public function init() {
		add_action( 'manage_users_extra_tablenav', array( $this, 'render_bulk_connect_control' ), 10000 );
		add_action( 'load-users.php', array( $this, 'process_bulk_connect_request' ) );
		add_action( 'show_user_profile', array( $this, 'render_profile_field' ) );
		add_action( 'edit_user_profile', array( $this, 'render_profile_field' ) );
		add_action( 'edit_user_profile_update', array( $this, 'save_profile_field' ), 10, 1 );
		add_action( 'profile_update', array( $this, 'save_profile_field' ), 10, 1 );
	}

	public function is_valid_registration_form_id( $form_id ) {
		$form_id = absint( $form_id );

		if ( 0 === $form_id ) {
			return false;
		}

		$post = get_post( $form_id );

		return $post && 'user_registration' === $post->post_type && 'publish' === $post->post_status;
	}
}
```

- [ ] **Step 4: Load the service from admin bootstrap**

In `includes/admin/class-ur-admin.php`, include the new file and call `init()` only in admin context.

```php
include_once __DIR__ . '/class-ur-admin-profile-connect.php';

if ( current_user_can( 'promote_users' ) ) {
	$profile_connect = new UR_Admin_Profile_Connect();
	$profile_connect->init();
}
```

- [ ] **Step 5: Run syntax and targeted tests**

Run:

```bash
php -l includes/admin/class-ur-admin-profile-connect.php
```

Expected: `No syntax errors detected`.

- [ ] **Step 6: Commit**

```bash
git add includes/admin/class-ur-admin-profile-connect.php includes/admin/class-ur-admin.php tests/php/admin/class-ur-admin-profile-connect-test.php
git commit -m "Add basic Profile Connect admin service"
```

---

## Task 2: Implement Bulk Connect And Disconnect On Users Table

**Files:**
- Modify: `includes/admin/class-ur-admin-profile-connect.php`
- Test: `tests/php/admin/class-ur-admin-profile-connect-test.php`

**Interfaces:**
- Consumes: `UR_Admin_Profile_Connect::is_valid_registration_form_id( int $form_id ): bool`.
- Produces: bulk Users table controls named `ur_profile_connect_form` and nonce `ur_profile_connect_nonce`.

- [ ] **Step 1: Write failing tests for bulk assignment**

```php
public function test_bulk_connect_updates_ur_form_id_for_selected_users() {
	$form_id = self::factory()->post->create(
		array(
			'post_type'   => 'user_registration',
			'post_status' => 'publish',
		)
	);
	$user_id = self::factory()->user->create();

	$this->set_current_user_with_capability( 'promote_users' );

	$_REQUEST['users']                  = array( $user_id );
	$_REQUEST['ur_profile_connect_form'] = (string) $form_id;
	$_REQUEST['ur_profile_connect_nonce'] = wp_create_nonce( 'ur-profile-connect-users' );

	$service = new UR_Admin_Profile_Connect();
	$result  = $service->process_bulk_connect_request( false );

	$this->assertSame( (string) $form_id, get_user_meta( $user_id, 'ur_form_id', true ) );
	$this->assertSame( 1, $result['connected'] );
}

public function test_bulk_disconnect_deletes_ur_form_id_for_selected_users() {
	$form_id = self::factory()->post->create(
		array(
			'post_type'   => 'user_registration',
			'post_status' => 'publish',
		)
	);
	$user_id = self::factory()->user->create();
	update_user_meta( $user_id, 'ur_form_id', $form_id );

	$this->set_current_user_with_capability( 'promote_users' );

	$_REQUEST['users']                   = array( $user_id );
	$_REQUEST['ur_profile_connect_form'] = '-1';
	$_REQUEST['ur_profile_connect_nonce'] = wp_create_nonce( 'ur-profile-connect-users' );

	$service = new UR_Admin_Profile_Connect();
	$result  = $service->process_bulk_connect_request( false );

	$this->assertSame( '', get_user_meta( $user_id, 'ur_form_id', true ) );
	$this->assertSame( 1, $result['disconnected'] );
}
```

- [ ] **Step 2: Add a test for invalid form rejection**

```php
public function test_bulk_connect_rejects_invalid_form_id() {
	$user_id = self::factory()->user->create();

	$this->set_current_user_with_capability( 'promote_users' );

	$_REQUEST['users']                   = array( $user_id );
	$_REQUEST['ur_profile_connect_form'] = '999999';
	$_REQUEST['ur_profile_connect_nonce'] = wp_create_nonce( 'ur-profile-connect-users' );

	$service = new UR_Admin_Profile_Connect();
	$result  = $service->process_bulk_connect_request( false );

	$this->assertSame( '', get_user_meta( $user_id, 'ur_form_id', true ) );
	$this->assertSame( 'invalid_form', $result['error'] );
}
```

- [ ] **Step 3: Render the Users table connect control**

Implementation requirements:

```php
public function render_bulk_connect_control( $which ) {
	if ( ! current_user_can( 'promote_users' ) ) {
		return;
	}

	$forms = ur_get_all_user_registration_form();
	$id    = 'bottom' === $which ? 'ur_profile_connect_form2' : 'ur_profile_connect_form';

	echo '<div class="alignleft actions">';
	echo '<label class="screen-reader-text" for="' . esc_attr( $id ) . '">' . esc_html__( 'Connect with form', 'user-registration' ) . '</label>';
	echo '<select name="ur_profile_connect_form" id="' . esc_attr( $id ) . '">';
	echo '<option value="">' . esc_html__( 'Connect with form...', 'user-registration' ) . '</option>';
	echo '<option value="-1">' . esc_html__( 'None', 'user-registration' ) . '</option>';

	foreach ( $forms as $form_id => $form_name ) {
		echo '<option value="' . esc_attr( $form_id ) . '">' . esc_html( $form_name ) . '</option>';
	}

	echo '</select>';
	wp_nonce_field( 'ur-profile-connect-users', 'ur_profile_connect_nonce' );
	submit_button( __( 'Connect', 'user-registration' ), '', 'ur_profile_connect_submit', false );
	echo '</div>';
}
```

- [ ] **Step 4: Implement processing with safe redirect**

Implementation requirements:

- Require selected `users`.
- Require non-empty `ur_profile_connect_form`.
- Verify `ur_profile_connect_nonce`.
- Require `current_user_can( 'promote_users' )`.
- Use `array_map( 'absint', ... )`.
- Use `delete_user_meta()` only for `-1`.
- Use `update_user_meta()` only for a valid published `user_registration` form.
- Redirect back to `users.php` with status counts after browser requests.
- Return a result array when called with `$redirect = false` for tests.

- [ ] **Step 5: Add admin notices for results**

Notices:

- Success: `Form connected for %d user(s).`
- Disconnect: `Form disconnected for %d user(s).`
- Error: `Form connection failed. Select a valid registration form and try again.`

- [ ] **Step 6: Verify in browser**

Manual browser check:

1. Open `wp-admin/users.php`.
2. Confirm `Connect with form...` appears beside the existing bulk actions.
3. Select one user with source `-`.
4. Select a registration form.
5. Click `Connect`.
6. Confirm success notice appears.
7. Confirm the user's `Source` column changes to the selected form.

- [ ] **Step 7: Commit**

```bash
git add includes/admin/class-ur-admin-profile-connect.php tests/php/admin/class-ur-admin-profile-connect-test.php
git commit -m "Add bulk Profile Connect assignment to Free"
```

---

## Task 3: Add Per-User Profile Assignment

**Files:**
- Modify: `includes/admin/class-ur-admin-profile-connect.php`
- Test: `tests/php/admin/class-ur-admin-profile-connect-test.php`

**Interfaces:**
- Consumes: `UR_Admin_Profile_Connect::is_valid_registration_form_id()`.
- Produces: a user profile field named `ur_profile_connect`.

- [ ] **Step 1: Write failing test for profile save**

```php
public function test_profile_save_updates_user_form_assignment() {
	$form_id = self::factory()->post->create(
		array(
			'post_type'   => 'user_registration',
			'post_status' => 'publish',
		)
	);
	$user_id = self::factory()->user->create();

	$this->set_current_user_with_capability( 'edit_users' );

	$_POST['ur_profile_connect']       = (string) $form_id;
	$_POST['ur_profile_connect_nonce'] = wp_create_nonce( 'ur-profile-connect-profile-' . $user_id );

	$service = new UR_Admin_Profile_Connect();
	$service->save_profile_field( $user_id );

	$this->assertSame( (string) $form_id, get_user_meta( $user_id, 'ur_form_id', true ) );
}
```

- [ ] **Step 2: Write failing test for disconnect**

```php
public function test_profile_save_disconnects_user_when_none_selected() {
	$form_id = self::factory()->post->create(
		array(
			'post_type'   => 'user_registration',
			'post_status' => 'publish',
		)
	);
	$user_id = self::factory()->user->create();
	update_user_meta( $user_id, 'ur_form_id', $form_id );

	$this->set_current_user_with_capability( 'edit_users' );

	$_POST['ur_profile_connect']       = '-1';
	$_POST['ur_profile_connect_nonce'] = wp_create_nonce( 'ur-profile-connect-profile-' . $user_id );

	$service = new UR_Admin_Profile_Connect();
	$service->save_profile_field( $user_id );

	$this->assertSame( '', get_user_meta( $user_id, 'ur_form_id', true ) );
}
```

- [ ] **Step 3: Render the profile field**

Add a `Connect To Form` row to user profile edit screens for users with `promote_users` or `edit_users` capability.

Requirements:

- Show current selected form from `get_user_meta( $user->ID, 'ur_form_id', true )`.
- Include `None` option.
- Include all forms from `ur_get_all_user_registration_form()`.
- Include nonce `ur_profile_connect_nonce`.

- [ ] **Step 4: Save the profile field**

Requirements:

- Return early if field is absent.
- Require `current_user_can( 'edit_user', $user_id )` or the existing project capability pattern for editing users.
- Verify nonce `ur-profile-connect-profile-$user_id`.
- Delete `ur_form_id` for `-1`.
- Reject invalid form IDs.
- Update `ur_form_id` for valid form IDs.

- [ ] **Step 5: Verify in browser**

Manual browser check:

1. Open `wp-admin/user-edit.php?user_id=<test user id>`.
2. Confirm `Connect To Form` appears.
3. Select a form and save.
4. Reopen the user profile.
5. Confirm the selected form remains selected.

- [ ] **Step 6: Commit**

```bash
git add includes/admin/class-ur-admin-profile-connect.php tests/php/admin/class-ur-admin-profile-connect-test.php
git commit -m "Add profile-level form assignment"
```

---

## Task 4: Replace Free Profile Connect Upsell With Limited Free Guidance

**Files:**
- Modify: `includes/admin/settings/class-ur-settings-registration-login.php`
- Modify: `includes/functions-ur-core.php`

**Interfaces:**
- Consumes: current settings page `registration_login` tab and `profile-connect` section.
- Produces: Free visible `Profile Connect` section explaining basic assignment and Pro advanced mapping.

- [ ] **Step 1: Add a settings rendering method**

In `class-ur-settings-registration-login.php`, add:

```php
public function get_profile_connect_settings() {
	return apply_filters(
		'user_registration_profile_connect_free_settings',
		array(
			'title'    => __( 'Profile Connect', 'user-registration' ),
			'sections' => array(
				'user_registration_profile_connect_basic' => array(
					'title'    => __( 'Connect existing users to registration forms', 'user-registration' ),
					'type'     => 'card',
					'desc'     => __( 'Assign existing WordPress users to a User Registration form from the Users screen or individual user profile.', 'user-registration' ),
					'settings' => array(),
				),
				'user_registration_profile_connect_pro' => array(
					'title'    => __( 'Advanced migration mapping', 'user-registration' ),
					'type'     => 'card',
					'desc'     => __( 'Upgrade to Pro to map external fields, external tables, and advanced migration sources.', 'user-registration' ),
					'settings' => array(),
				),
			),
		)
	);
}
```

- [ ] **Step 2: Route `profile-connect` to the Free settings method**

Change:

```php
if ( in_array( $current_section, [ 'social-connect', 'profile-connect', 'popup', 'invite-code', 'file-upload', 'role-based-redirection' ] ) ) {
	return $this->upgrade_to_pro_setting();
}
```

To:

```php
if ( 'profile-connect' === $current_section ) {
	return $this->get_profile_connect_settings();
}

if ( in_array( $current_section, [ 'social-connect', 'popup', 'invite-code', 'file-upload', 'role-based-redirection' ], true ) ) {
	return $this->upgrade_to_pro_setting();
}
```

- [ ] **Step 3: Update upsell metadata**

In `includes/functions-ur-core.php`, update Profile Connect upsell copy so it no longer claims the whole feature is paid.

Recommended copy:

- Excerpt: `Map existing user data during advanced migrations.`
- Description:
  - `Map external fields to User Registration fields`
  - `Import values from supported external data sources`

- [ ] **Step 4: Verify in browser**

Manual browser check:

1. Open `wp-admin/admin.php?page=user-registration-settings&tab=registration_login&section=profile-connect`.
2. Confirm the section is no longer a generic locked addon card.
3. Confirm basic assignment is described as available in Free.
4. Confirm Pro copy is limited to advanced mapping/migration.

- [ ] **Step 5: Commit**

```bash
git add includes/admin/settings/class-ur-settings-registration-login.php includes/functions-ur-core.php
git commit -m "Update Profile Connect settings for Free assignment"
```

---

## Task 5: Rewrite Existing-User Notices And Fallback Admin Copy

**Files:**
- Modify: `includes/admin/class-ur-admin-user-list-manager.php`
- Modify: `templates/myaccount/form-edit-profile-non-urm-user.php`

**Interfaces:**
- Consumes: existing non-URM user count logic.
- Produces: copy that points admins to the Free assignment workflow instead of selling basic Profile Connect.

- [ ] **Step 1: Update admin notice copy**

Current behavior links to the paid Profile Connect feature page when 5 or more non-URM users exist.

Replace with copy like:

```php
printf(
	/* translators: %s: number of existing users */
	esc_html__( 'Your site has %s users registered before this plugin. You can connect them to a registration form from Users > All Users.', 'user-registration' ),
	'<span class="highlight">' . intval( $existing_non_urm_users ) . '</span>'
);
```

Add an admin link to `users.php` if the UI pattern allows it.

- [ ] **Step 2: Update My Account fallback admin copy**

Replace the admin-only Profile Connect upsell with guidance:

```php
esc_html_e( 'Connect existing users to a registration form from Users > All Users so they can use form-based profile fields.', 'user-registration' );
```

- [ ] **Step 3: Verify in browser**

Manual browser check:

1. Open `wp-admin/users.php`.
2. Confirm notice no longer says to buy Profile Connect for basic assignment.
3. Open My Account edit profile as an admin user without `ur_form_id`.
4. Confirm admin-facing fallback copy points to the Free assignment workflow.

- [ ] **Step 4: Commit**

```bash
git add includes/admin/class-ur-admin-user-list-manager.php templates/myaccount/form-edit-profile-non-urm-user.php
git commit -m "Rewrite existing user Profile Connect notices"
```

---

## Task 6: Keep Advanced Mapping In Pro/Add-on And Prevent Duplicate Hooks

**Files:**
- Modify: `../user-registration-profile-connect/includes/class-user-registration-profile-connect.php`
- Modify: `../user-registration-profile-connect/includes/class-user-registration-profile-connect-process.php`
- Modify: `../user-registration-profile-connect/includes/settings/class-user-registration-profile-connect-settings.php`

**Interfaces:**
- Consumes: Free plugin basic assignment availability.
- Produces: addon behavior that avoids duplicate basic UI and keeps advanced mapping active.

- [ ] **Step 1: Add a Free capability flag**

In the Free plugin, define a capability function or constant:

```php
if ( ! function_exists( 'ur_has_basic_profile_connect' ) ) {
	function ur_has_basic_profile_connect() {
		return true;
	}
}
```

- [ ] **Step 2: Gate addon basic hooks**

In the addon process constructor, wrap basic hooks:

```php
if ( ! function_exists( 'ur_has_basic_profile_connect' ) || ! ur_has_basic_profile_connect() ) {
	add_action( 'manage_users_extra_tablenav', array( $this, 'profile_connect' ), 10000 );
	add_action( 'load-users.php', array( $this, 'profile_connect_process' ) );
	add_action( 'show_user_profile', array( $this, 'render_profile_field' ) );
	add_action( 'edit_user_profile', array( $this, 'render_profile_field' ) );
	add_action( 'edit_user_profile_update', array( $this, 'save_profile_field' ), 1, 1 );
	add_action( 'profile_update', array( $this, 'save_profile_field' ), 1, 1 );
}
```

Keep the advanced settings registration active for Pro/addon users.

- [ ] **Step 3: Verify duplicate prevention**

Manual browser check with addon active:

1. Activate `User Registration Profile Connect`.
2. Open `wp-admin/users.php`.
3. Confirm there is only one connect-to-form control.
4. Open user profile edit.
5. Confirm there is only one `Connect To Form` field.
6. Open Profile Connect settings.
7. Confirm advanced mapping settings remain available where intended.

- [ ] **Step 4: Commit**

```bash
git add includes/functions-ur-core.php ../user-registration-profile-connect/includes/class-user-registration-profile-connect.php ../user-registration-profile-connect/includes/class-user-registration-profile-connect-process.php ../user-registration-profile-connect/includes/settings/class-user-registration-profile-connect-settings.php
git commit -m "Prevent duplicate Profile Connect addon hooks"
```

---

## Task 7: Verification And Release Readiness

**Files:**
- No new production files unless verification finds defects.

**Interfaces:**
- Consumes all previous tasks.
- Produces verified implementation ready for PR/review.

- [ ] **Step 1: Run PHP syntax checks**

Run:

```bash
php -l includes/admin/class-ur-admin-profile-connect.php
php -l includes/admin/settings/class-ur-settings-registration-login.php
php -l includes/admin/class-ur-admin-user-list-manager.php
php -l templates/myaccount/form-edit-profile-non-urm-user.php
```

Expected: `No syntax errors detected` for each file.

- [ ] **Step 2: Run automated tests**

Run the repository's PHP test command for:

- `tests/php/admin/class-ur-admin-profile-connect-test.php`
- Any existing Users/admin settings tests impacted by the change.

Expected: all targeted tests pass.

- [ ] **Step 3: Run static checks**

Run the repository's coding standards command if available. If not available, run the closest configured lint/static analysis command and document the limitation.

- [ ] **Step 4: Browser verify Free without addon active**

Checks:

- Users table shows one connect-to-form control.
- Bulk connect writes `ur_form_id`.
- Bulk disconnect deletes `ur_form_id`.
- User profile edit assignment works.
- Profile Connect settings section shows Free assignment guidance.
- Non-URM notice no longer sells basic Profile Connect.

- [ ] **Step 5: Browser verify with addon active**

Checks:

- No duplicate connect controls.
- Advanced mapping settings remain available.
- External mapping is not presented as Free.

- [ ] **Step 6: Regression check direct consumers**

Inspect these known direct `ur_form_id` consumers and confirm no scope-breaking behavior changed:

- `includes/class-ur-email-approval.php`
- `includes/class-ur-shortcodes.php`
- `includes/admin/class-ur-admin-export-users.php`
- `includes/admin/class-ur-admin-user-list-manager.php`
- `includes/Analytics/Services/AnalyticsDataService.php`
- Pro analytics/payment/frontend-listing consumers in `user-registration-pro`

Expected: assignment writes the same meta key they already consume.

- [ ] **Step 7: Prepare PR description**

Include:

- Summary of Free basic Profile Connect assignment.
- What remains Pro.
- Exact verification commands and browser checks.
- Note that default-form fallback is not included.
- Note duplicate-addon handling.

- [ ] **Step 8: Final commit if verification fixes were needed**

```bash
git add <changed-files>
git commit -m "Verify Profile Connect free tier readiness"
```

---

## Self-Review

**Spec coverage:** Covered current UI baseline, limited Free scope, Pro retained scope, issue alignment, notices, addon coexistence, and verification.

**Placeholder scan:** No placeholder tasks are intentionally left. Any project-specific test command must be replaced by the real repository command during implementation discovery if the exact command is not already known.

**Type consistency:** The central service name is consistently `UR_Admin_Profile_Connect`. The central validation method is consistently `is_valid_registration_form_id( $form_id )`.

**Review Focus coverage:** The five review-focus cases are covered across Tasks 1, 2, 3, 6, and 7.
