---
layout: post
title: dynamic page path with a custom template
categories: sin
tags: [php, wordpress]
---

Wordpress is great if you have data directly in your database, but what happens when you want to create pages from a remote API or data feed, and you want to load all the information dynamically on a page? Things get a little tricky, you'll have to stray from the beaten path, but there is a way to build it!

## What we'll cover
* How to create an endpoint (rewrite rule)
* How to capture and parse URI data
* How to serve up a custom page template
* How to package it up

## Example
In this example, we will build an endpoint that creates pages dynamically based on a literary ISBN number. We will want to accept requests coming in at `/book/detail/ISBN-10`, and then pull data from the [Open Library API](https://openlibrary.org/dev/docs/api/books){:target="_blank"}.
## How to create an endpoint
1. Create a transient. This will help us leverage the transient API, and easily cache our data.
```php
<?php
function activate() {
        set_transient( 'our_plugin_name_flush', 1, 60 );
    }
```
2. Create a rewrite rule. This will allow WordPress to detect the custom URL path. The first value passed into the [add_rewrite_rule](https://codex.wordpress.org/Rewrite_API/add_rewrite_rule){:target="_blank"} function is the regex we will use to match our path. The second input is the page name that we will route to. It is good to note that we want it to hit the `index.php` file, so Wordpress knows what's up. The final input value will tell Wordpress where to place the rule on the list of rewrite rules. In this case, I want to make my rewrite gets matched before any default WordPress rule. We're also going to throw some logic into this to find cached transients and wipe them to ensure the page loads with our new rule.
```php
<?php
public function rewrite() {
        add_rewrite_rule('^book\/detail\/(\d{10})\/?', 'index.php?pagename=book/detail/$matches[1]', 'top');
        if (get_transient('our_plugin_name_flush')) {
            delete_transient('our_plugin_name_flush');
            flush_rewrite_rules();
        }
    }
```
3. Set the page HTTP response. By default, wordpress looks up your page request in the site `_posts` table, and then generates the appropriate HTTP response. Since we are not using the post system, we need to trick wordpress into returning a 200 response when our page regex matches.
```php
<?php
public function change_http_response() {
        $uri = $_SERVER["REQUEST_URI"];
        if (preg_match('/\/book\/detail\/(\d{10})\/?', $uri)) {
            status_header(200);
        }
    }
```
4. Set custom page template response. First, we will match the request URI for our regex pattern, and then we will double check that our specified template exists. If it does, we will ensure we return a 200 response and return our template. If we cannot find it, we will fallback to the index request.
```php
<?php
public function change_template($template) {
        $uri = $_SERVER["REQUEST_URI"];
        if (preg_match('/\/book\/detail\/(\d{10})\/?/', $uri)) {
            //Check plugin directory
            $newTemplate = IPG_PATH . 'includes/templates/book.php';
            if (file_exists($newTemplate)) {
                // Set http response
                global $wp_query;
                if ($wp_query->is_404) {
                    $wp_query->is_404 = false;
                }
                // Ensure 200 response
                status_header(200);

                return $newTemplate;
            }
        }
        //Fall back to original template
        return $template;
    }
```
5. Create constructor. It is omportant to note that the `change_http_response` function has to be run at init otherwise, search bots will recieve a 404, but normal user agents will still get a 200. This took me a long time to figure out, because I initially only set the http response in the `change_template` funtion that gets called at `template_include`. Best to let the bots crawl your page for SEO.
```php
<?php
public function __construct() {
        register_activation_hook(__FILE__, array($this, 'activate'));
        add_action('init', array($this, 'rewrite'));
        add_action('init', array($this, 'change_http_response'));
        add_action('template_include', array($this, 'change_template'));
    }
```
6. Combine it. `ipg-routing.php`
```php
<?php
    namespace ISBN_PAGE_GENERATOR;
    class IPGRouting {
        public function __construct() {
            register_activation_hook(__FILE__, array($this, 'activate'));
            add_action('init', array($this, 'rewrite'));
            add_action('init', array($this, 'change_http_response'));
            add_action('template_include', array($this, 'change_template'));

        }
        function activate() {
            set_transient( 'our_plugin_name_flush', 1, 60 );
        }
        public function rewrite() {
            add_rewrite_rule('^book\/detail\/(\d{10})\/?', 'index.php?pagename=book/detail/$matches[1]', 'top');
            if (get_transient('our_plugin_name_flush')) {
                delete_transient('our_plugin_name_flush');
                flush_rewrite_rules();
            }
        }
        public function change_http_response() {
            $uri = $_SERVER["REQUEST_URI"];
            if (preg_match('/\/book\/detail\/(\d{10})\/?', $uri)) {
                status_header(200);
            }
        }
        public function change_template($template) {
            $uri = $_SERVER["REQUEST_URI"];
            if (preg_match('/\/book\/detail\/(\d{10})\/?/', $uri)) {
                //Check plugin directory
                $newTemplate = IPG_PATH . 'includes/templates/book.php';
                if (file_exists($newTemplate)) {
                    // Set http response
                    global $wp_query;
                    if ($wp_query->is_404) {
                        $wp_query->is_404 = false;
                    }
                    // Ensure 200 response
                    status_header(200);
                    return $newTemplate;
                }
            }
            //Fall back to original template
            return $template;
        }
    }
    new IPGRouting;
```

## How to serve up a custom page template and parse the URI
Without spending too much time here, the main concept is that we will use `preg_match` to pull the ISBN from the URI. We will then make a call to the Open Library API, and pull all of the book's data in JSON format. We will also add a function to return a 404 if the API call returns an empty response (the ISBN number doesn't exist). Now you have the data in the page, and can leverage a JS templating engine like Mustache.js, or convert the JSON to a PHP array and assign them to variables and manually generate the page layout. In my experience, JS templating provides much fast page load times.
```php
<?php
/**
 * Main template for displaying book pages
 *
 * @package Your Theme Package
 */
// Get ISBN
    $uri = $_SERVER["REQUEST_URI"];
    preg_match('/\/book\/detail\/(\d{10})/', $uri, $isbn_10);
// Output ISBN Match
    global $page_isbn_10;
    $page_isbn_10 = $isbn_10[1];
// Get Data
    $request = file_get_contents("http://openlibrary.org/api/books?bibkeys=ISBN:{$page_isbn_10}&format=json&jscmd=details&preview=full");
    $json_request = json_decode($request);
// Invalid ISBN return 404
    if ($json_request == {}){
      status_header( 404 );
      get_template_part( 404 ); exit();
    }
// Custom Title and Description
    add_filter( 'wp_title', __NAMESPACE__. '\ipg_custom_title', 10, 3 );
    function ipg_custom_title($title){
        global $json_request;
        $title = $json_request[key($json_request)]->details->title;
        return $title;
    };

// More default header stuff
    get_header();
    get_template_part( 'menu', 'index' ); //the  menu + logo/site title ?>

<div class="sixteen columns alpha">
    <div id="primary">
        <div id="content">
            <div class="main">
                <header class="entry-header">
                    <h1 class="entry-title"><?php ?></h1>
                </header>

                <?php the_post(); ?>
                <article id="post-<?php the_ID(); ?>" <?php post_class(); ?>>
                    <div class="entry-content">
                        <!-- Begin Book -->
                        <div id="book">
                        <!-- USE TEMPLATING ENGINE TO RENDER DATA -->
                        <!-- mustache.js is a great option -->
                        <!-- https://github.com/janl/mustache.js -->
                        </div>
                        <!-- End Book -->
                        <?php edit_post_link( __( 'Edit', 'appfolio' ), '<span class="edit-link">', '</span>' ); ?>
                    </div><!-- .entry-content -->
                </article><!-- #post-<?php the_ID(); ?> -->

            </div><!-- #main -->
        </div><!-- #content -->
    </div><!-- #primary -->
</div>

<?php get_footer(); ?>
```


## How to package it up
- /isbn-page-generator
    - isbn-page-generator.php
    - /includes
        - ipg-routing.php
        - /templates
            - book.php
            - /assets

1. Create plugin root. `isbn-page-generator.php`
```php
<?php
    namespace ISBN_PAGE_GENERATOR;
    /*
    Plugin Name: 
    Description: Generates all the items needed dynamic JSON pages
    Version: 1.0
    Author: Will Sizoo + Lazy Ruby
    Author URI: http://lazyruby.com
    */
    // Useful global constants
	    define('IPG_URL', plugin_dir_url(__FILE__));
	    define('IPG_PATH', dirname(__FILE__) . '/');
    // Loads Plugin
	    function loader() {
	        require_once __DIR__ . '/includes/ipg-routing.php';
	    }
	    add_action('plugins_loaded', __NAMESPACE__ . '\loader');
```
2. Organize the plugin. Don't build things in a million separate files, but also be mindful that keeping code that has isolated function cleanly formatted in a separate file can be beneficial. Start off with a sketch of your structure, but be willing to change along the way. Think about what will make life easier for future you. If it's for someone else, fuck it, be lazy.