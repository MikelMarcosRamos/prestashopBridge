prestashopBridge
================

Allow Prestashop to work nicely with your existing PHP Application (Drupal, Symfony, Joomla, Wordpress, ...).

Use your authentication mechansim of choice with prestashop (SSO, Oauth, 2-factor, ... )

Benefits:

 - customers don't have to create an account in Prestashop
 - customers authenticate only in your application


Features:

 - Login / Logout customers
 - Reset user password
 - Create customers
 - Create carts


Requirements:

 - should be installed in the same server than prestashop (because some  Prestashop code is loaded with _include()_  )
 - should be served from the same domain:port than Prestashop (because of the auth cookie)



En Français
=====

Ce projet vous est utile si vous voulez que :

- vos clients n'aient qu'un seul mot de passe pour votre site et pour votre boutique
- vos que vos clients ne s'authentifient que par votre site
- utiliser une autre méthode d'authentification (2-facteurs, OAuth, CAS, ...) que celles proposent par Prestashop.


Fonctionnalités :

- connecter un client
- déconnecter un client
- changement de mot de passe du client synchronisé vers le PrestaShop
- créer des clients dans la base de prestashop
- créer des commandes (optionnellement avec des références)

Contraintes :

- Doit être installé sur le même serveur que Prestashop (car utilise quelques fonctions internes de prestashop - via _include()_ )
- Doit être servi depuis le même nom de domaine / port que Prestashop (à cause du cookie d'authentification)


Installation
====

In your composer.json:

```json
{
	"require": {
		"mahjouba91/prestashop-bridge": "dev-master"
	},
	"repositories": [{
		"type": "vcs",
		"url": "https://github.com/Mahjouba91/prestashopBridge.git"
	}]
}
```


Example
=====

In a symfony application:

```php
<?php 

namespace Acme\DemoBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Symfony\Component\Security\Core\Exception\AccessDeniedException;

use Mahjouba91\PrestashopBridge\PrestashopBridge;

class PrestashopBridgeExampleController extends Controller
{
	public function loginAction()
	{
		if (!$this->get('security.context')->isGranted('ROLE_USER')) {
			throw new AccessDeniedException();
		}

		$prestaBridge = new PrestashopBridge('/path/to/prestashop/', 1);

		$user = $this->getUser(); //get connected user

		if (!$prestaBridge->userExist($user->getEmail())) //if user exist in prestahop database
			$prestaBridge->createUser(
				$user->getEmail(),
				$user->getLastName(),
				$user->getFirstName()
			);

		$prestaBridge->login($user->getEmail());

		return $this->redirect('http://prestashop_url/');
	}
}

```

In a WordPress plugin:

```php
<?php
/**
* Plugin Name: Prestashop Bridge
* Description: Handle registration/login between PrestaShop and WordPress
* Version: 1.0
* Author: Florian TIAR
* Author URI: http://wpstrategie.fr
* License: GPL2 license
*/

// don't load directly
if ( ! defined( 'ABSPATH' ) ) {
	die( '-1' );
}

require( ABSPATH . '/vendor/autoload.php' );
use Mahjouba91\PrestashopBridge\PrestashopBridge;

// Hook after a new user is registred
add_action( 'user_register', function( $user_id ) {
	// If it's a front registration, wait the email validation because the user don't choose any password yet
	if ( ! isset( $_POST['pass1'] ) ) {
		return;
	}
	// For backend registration, create prestashop account now
	cbl_add_customer( $_POST, $user_id );
}, 10, 1 );

// After password reset
add_action( 'after_password_reset', function( $user, $new_pass ) {
	$presta_path = cbl_get_prestashop_path();
	$prestaBridge = new PrestashopBridge( $presta_path, 1 );
	if ( $prestaBridge->userExist( $user->user_email ) ) {
		// The user want to update its password, do it also on PrestaShop
		$prestaBridge->updateUserPassword( $user->user_email, $new_pass );
	} else {
		// It's a new user, so create it on PrestaShop
		$prestaBridge->createUser( $user->user_email, $user->last_name, $user->first_name, $new_pass );
	}
}, 10, 2 );

// After WordPress login, login the user in PrestaShop too
add_action( 'wp_login', function( $user_login, $user ) {
	$logger = new Bea_Log( WP_CONTENT_DIR . '/presta' );
	$logger->log_this( 'wp_login hook', Bea_Log::gravity_0 );

	$presta_path = cbl_get_prestashop_path();
	$prestaBridge = new PrestashopBridge( $presta_path, 1 );
	$login = $prestaBridge->login( $user->user_email );
	$logger->log_this( 'login : ' . $login, Bea_Log::gravity_0 );

}, 10, 2 );

// After WordPress logout, logout the user in PrestaShop too
add_action( 'wp_logout', function() {
	$logger = new Bea_Log( WP_CONTENT_DIR . '/presta' );
	$logger->log_this( 'wp_logout hook', Bea_Log::gravity_0 );

	$presta_path = cbl_get_prestashop_path();
	$prestaBridge = new PrestashopBridge( $presta_path, 1 );
	$prestaBridge->logout();
});

function cbl_add_customer( $params, $user_id ) {
	$user = get_userdata( $user_id );
	$presta_path = cbl_get_prestashop_path();
	$prestaBridge = new PrestashopBridge( $presta_path, 1 );
	if ( $prestaBridge->userExist( $user->user_email ) ) {
		return false;
	}
	
	$prestaBridge->createUser( $user->user_email, $user->last_name, $user->first_name, $params['pass1'] );
	return true;
}

function cbl_get_prestashop_path () {
	return realpath( ABSPATH . 'shop/' );
}


```
