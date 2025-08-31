# Build v1.2.0: screenshot-style UI, fully responsive, A–Z chips, dark CTA, credits "Created by Mulaudzi H".
import os, zipfile, textwrap, json

base = "/mnt/data/forex-brokers-directory-v1.2.0"
assets = os.path.join(base, "assets")
os.makedirs(assets, exist_ok=True)

php = textwrap.dedent(r'''<?php
/**
 * Plugin Name: Forex Brokers Directory
 * Description: Screenshot-style, fully responsive brokers directory with filters, A–Z chips, dark CTA, CSV import, shortlinks (/go/{broker}), pagination & sorting. Uses Featured Image as logo (from Media). All-in-one, free.
 * Version: 1.2.0
 * Author: Mulaudzi H
 * License: GPLv2 or later
 * Text Domain: forex-brokers-directory
 */

if ( ! defined( 'ABSPATH' ) ) { exit; }

class Forex_Brokers_Directory {
    private $opt_key = 'fbd_settings';

    public function __construct() {
        register_activation_hook(__FILE__, [$this, 'activate']);
        register_deactivation_hook(__FILE__, [$this, 'deactivate']);

        add_action('init', [$this, 'register_post_type_and_taxonomies']);
        add_action('init', [$this, 'register_shortlink_rewrite']);
        add_action('template_redirect', [$this, 'handle_shortlink_redirect']);
        add_action('add_meta_boxes', [$this, 'add_meta_boxes']);
        add_action('save_post', [$this, 'save_meta_boxes']);
        add_shortcode('forex_brokers', [$this, 'shortcode_directory']);
        add_shortcode('forex_brokers_table', [$this, 'shortcode_table']);
        add_action('wp_enqueue_scripts', [$this, 'enqueue_assets']);

        add_filter('manage_broker_posts_columns', [$this, 'admin_columns']);
        add_action('manage_broker_posts_custom_column', [$this, 'admin_columns_render'], 10, 2);

        add_action('admin_menu', [$this, 'admin_menu']);
        add_action('admin_post_fbd_import_csv', [$this, 'handle_import_csv']);

        add_theme_support('post-thumbnails', ['broker']);
    }

    public function activate(){ 
        $this->register_post_type_and_taxonomies(); 
        $this->register_shortlink_rewrite();
        flush_rewrite_rules(); 
        if(!get_option($this->opt_key)){
            add_option($this->opt_key, [
                'cta_text' => 'Visit Broker',
                'cta_color' => '#111827',
                'cta_hover' => '#0b1220',
                'per_page' => '12',
                'show_credit' => '1',
                'credit_text' => 'Created by Mulaudzi H',
            ]);
        }
    }
    public function deactivate(){ flush_rewrite_rules(); }

    public function plugin_url($path=''){ return plugins_url($path, __FILE__); }

    /* === Data Model === */
    public function register_post_type_and_taxonomies() {
        register_post_type('broker', [
            'labels' => [
                'name' => __('Brokers','forex-brokers-directory'),
                'singular_name' => __('Broker','forex-brokers-directory'),
                'add_new_item' => __('Add New Broker','forex-brokers-directory'),
                'edit_item' => __('Edit Broker','forex-brokers-directory'),
            ],
            'public' => true,
            'menu_icon' => 'dashicons-chart-line',
            'supports' => ['title','editor','thumbnail','excerpt'],
            'has_archive' => true,
            'rewrite' => ['slug' => 'brokers', 'with_front' => false],
            'show_in_rest' => true,
        ]);

        register_taxonomy('broker_country', 'broker', [
            'label' => __('Country','forex-brokers-directory'),
            'public' => true,
            'hierarchical' => false,
            'rewrite' => ['slug' => 'broker-country'],
            'show_in_rest' => true,
        ]);
        register_taxonomy('broker_platform', 'broker', [
            'label' => __('Platforms','forex-brokers-directory'),
            'public' => true,
            'hierarchical' => false,
            'rewrite' => ['slug' => 'broker-platform'],
            'show_in_rest' => true,
        ]);
        register_taxonomy('broker_account', 'broker', [
            'label' => __('Account Types','forex-brokers-directory'),
            'public' => true,
            'hierarchical' => false,
            'rewrite' => ['slug' => 'broker-account'],
            'show_in_rest' => true,
        ]);
    }

    /* === Shortlink /go/{broker} === */
    public function register_shortlink_rewrite(){
        add_rewrite_rule('^go/([^/]+)/?$', 'index.php?fbd_go=$matches[1]', 'top');
        add_rewrite_tag('%fbd_go%', '([^&]+)');
    }
    public function handle_shortlink_redirect(){
        $slug = get_query_var('fbd_go');
        if(!$slug) return;
        $post = get_page_by_path(sanitize_title($slug), OBJECT, 'broker');
        if($post){
            $url = get_post_meta($post->ID, '_affiliate_url', true);
            if($url){
                wp_redirect( esc_url($url), 302 );
                exit;
            }
        }
        status_header(404);
        nocache_headers();
        include get_query_template('404');
        exit;
    }

    /* === Metabox === */
    public function add_meta_boxes() {
        add_meta_box('broker_details', __('Broker Details','forex-brokers-directory'), [$this,'render_meta_box'], 'broker', 'normal', 'default');
    }
    public function render_meta_box($post) {
        wp_nonce_field('broker_details_nonce', 'broker_details_nonce');
        $fields = [
            'min_deposit' => get_post_meta($post->ID, '_min_deposit', true),
            'rating' => get_post_meta($post->ID, '_rating', true),
            'spread_from' => get_post_meta($post->ID, '_spread_from', true),
            'leverage' => get_post_meta($post->ID, '_leverage', true),
            'affiliate_url' => get_post_meta($post->ID, '_affiliate_url', true),
        ]; ?>
        <style>.fbd-field{margin-bottom:12px}.fbd-field label{display:block;font-weight:600;margin-bottom:6px}.fbd-field input[type="text"],.fbd-field input[type="number"]{width:100%;max-width:420px}.fbd-help{font-size:12px;opacity:.7}</style>
        <div class="fbd-field"><label><?php _e('Minimum Deposit (USD)','forex-brokers-directory'); ?></label><input type="number" step="0.01" name="min_deposit" value="<?php echo esc_attr($fields['min_deposit']); ?>"></div>
        <div class="fbd-field"><label><?php _e('Rating (0–5, e.g., 4.5)','forex-brokers-directory'); ?></label><input type="number" step="0.1" min="0" max="5" name="rating" value="<?php echo esc_attr($fields['rating']); ?>"></div>
        <div class="fbd-field"><label><?php _e('Spread From (e.g., 0.6 pips)','forex-brokers-directory'); ?></label><input type="text" name="spread_from" value="<?php echo esc_attr($fields['spread_from']); ?>"></div>
        <div class="fbd-field"><label><?php _e('Leverage (e.g., 1:500)','forex-brokers-directory'); ?></label><input type="text" name="leverage" value="<?php echo esc_attr($fields['leverage']); ?>"></div>
        <div class="fbd-field"><label><?php _e('Affiliate / Sign-Up URL','forex-brokers-directory'); ?></label><input type="text" name="affiliate_url" value="<?php echo esc_attr($fields['affiliate_url']); ?>"><div class="fbd-help"><?php _e('Pretty shortlink: /go/{broker-slug}','forex-brokers-directory'); ?></div></div>
        <p class="fbd-help"><?php _e('Logo: set the broker logo as the Featured Image (from Media Library).','forex-brokers-directory'); ?></p>
    <?php }
    public function save_meta_boxes($post_id) {
        if ( ! isset($_POST['broker_details_nonce']) || ! wp_verify_nonce($_POST['broker_details_nonce'],'broker_details_nonce') ) return;
        if ( defined('DOING_AUTOSAVE') && DOING_AUTOSAVE ) return;
        if ( ! current_user_can('edit_post', $post_id) ) return;
        foreach (['min_deposit','rating','spread_from','leverage','affiliate_url'] as $f) {
            $val = isset($_POST[$f]) ? sanitize_text_field($_POST[$f]) : '';
            update_post_meta($post_id, "_{$f}", $val);
        }
    }

    /* === Assets === */
    public function enqueue_assets() {
        if ( is_singular() || is_post_type_archive() ) {
            global $post;
            if ( isset($post->post_content) && ( has_shortcode($post->post_content, 'forex_brokers') || has_shortcode($post->post_content, 'forex_brokers_table') ) ) {
                $opt = get_option($this->opt_key);
                wp_enqueue_style('fbd-style', $this->plugin_url('assets/style.css'), [], '1.2.0');
                $vars = ':root{--cta:'.esc_attr($opt['cta_color']).';--cta-hover:'.esc_attr($opt['cta_hover']).';}';
                if(!empty($opt['credit_text'])){ $vars .= '.fbd-credit:after{content:" '.esc_js($opt['credit_text']).'";}'; }
                wp_add_inline_style('fbd-style', $vars);
                wp_enqueue_script('fbd-script', $this->plugin_url('assets/script.js'), [], '1.2.0', true);
            }
        }
    }

    /* === Admin: Settings + CSV Import === */
    public function admin_menu(){
        add_submenu_page('edit.php?post_type=broker', __('Directory Settings','forex-brokers-directory'), __('Settings','forex-brokers-directory'), 'manage_options', 'fbd-settings', [$this,'render_settings_page']);
        add_submenu_page('edit.php?post_type=broker', __('CSV Import','forex-brokers-directory'), __('CSV Import','forex-brokers-directory'), 'manage_options', 'fbd-import', [$this,'render_import_page']);
    }

    public function render_settings_page(){
        if(isset($_POST['fbd_save']) && check_admin_referer('fbd_save_opts')){
            $opt = [
                'cta_text' => sanitize_text_field($_POST['cta_text'] ?? 'Visit Broker'),
                'cta_color' => sanitize_text_field($_POST['cta_color'] ?? '#111827'),
                'cta_hover' => sanitize_text_field($_POST['cta_hover'] ?? '#0b1220'),
                'per_page' => sanitize_text_field($_POST['per_page'] ?? '12'),
                'show_credit' => isset($_POST['show_credit']) ? '1' : '0',
                'credit_text' => sanitize_text_field($_POST['credit_text'] ?? 'Created by Mulaudzi H'),
            ];
            update_option($this->opt_key, $opt);
            echo '<div class="updated"><p>'.esc_html__('Saved.','forex-brokers-directory').'</p></div>';
        }
        $opt = get_option($this->opt_key);
        ?>
        <div class="wrap">
            <h1><?php _e('Forex Brokers Directory Settings','forex-brokers-directory'); ?></h1>
            <form method="post">
                <?php wp_nonce_field('fbd_save_opts'); ?>
                <table class="form-table">
                    <tr><th><label><?php _e('Default CTA Text','forex-brokers-directory'); ?></label></th><td><input type="text" name="cta_text" value="<?php echo esc_attr($opt['cta_text']); ?>"></td></tr>
                    <tr><th><label><?php _e('CTA Color','forex-brokers-directory'); ?></label></th><td><input type="text" name="cta_color" value="<?php echo esc_attr($opt['cta_color']); ?>" class="regular-text"></td></tr>
                    <tr><th><label><?php _e('CTA Hover Color','forex-brokers-directory'); ?></label></th><td><input type="text" name="cta_hover" value="<?php echo esc_attr($opt['cta_hover']); ?>" class="regular-text"></td></tr>
                    <tr><th><label><?php _e('Cards per page','forex-brokers-directory'); ?></label></th><td><input type="number" name="per_page" value="<?php echo esc_attr($opt['per_page']); ?>" min="3" max="60"></td></tr>
                    <tr><th><label><?php _e('Show footer credit','forex-brokers-directory'); ?></label></th><td><label><input type="checkbox" name="show_credit" <?php checked($opt['show_credit'],'1'); ?>> <?php _e('Yes','forex-brokers-directory'); ?></label></td></tr>
                    <tr><th><label><?php _e('Credit text','forex-brokers-directory'); ?></label></th><td><input type="text" name="credit_text" value="<?php echo esc_attr($opt['credit_text']); ?>"></td></tr>
                </table>
                <p><button class="button button-primary" name="fbd_save" value="1"><?php _e('Save Settings','forex-brokers-directory'); ?></button></p>
            </form>
        </div>
        <?php
    }

    public function render_import_page(){
        ?>
        <div class="wrap">
            <h1><?php _e('CSV Import Brokers','forex-brokers-directory'); ?></h1>
            <p><?php _e('CSV columns: name, country (comma), platforms (comma), accounts (comma), min_deposit, rating, spread_from, leverage, affiliate_url, logo_url','forex-brokers-directory'); ?></p>
            <form method="post" enctype="multipart/form-data" action="<?php echo esc_url(admin_url('admin-post.php')); ?>">
                <?php wp_nonce_field('fbd_import_csv'); ?>
                <input type="hidden" name="action" value="fbd_import_csv">
                <input type="file" name="csv" accept=".csv" required>
                <p><button class="button button-primary"><?php _e('Import','forex-brokers-directory'); ?></button></p>
            </form>
        </div>
        <?php
    }

    public function handle_import_csv(){
        if ( ! current_user_can('manage_options') ) wp_die('Permission denied');
        check_admin_referer('fbd_import_csv');
        if ( empty($_FILES['csv']['tmp_name']) ) wp_die('No file');

        $fh = fopen($_FILES['csv']['tmp_name'], 'r');
        if(!$fh) wp_die('Cannot open CSV');

        $header = fgetcsv($fh);
        $map = array_flip($header);
        require_once ABSPATH.'wp-admin/includes/media.php';
        require_once ABSPATH.'wp-admin/includes/file.php';
        require_once ABSPATH.'wp-admin/includes/image.php';

        $count = 0;
        while(($row = fgetcsv($fh)) !== false){
            $name = sanitize_text_field($row[$map['name']] ?? '');
            if(!$name) continue;
            $post_id = wp_insert_post([
                'post_type' => 'broker',
                'post_status' => 'publish',
                'post_title' => $name,
            ]);
            if(is_wp_error($post_id)) continue;

            foreach([
                'broker_country' => 'country',
                'broker_platform' => 'platforms',
                'broker_account' => 'accounts',
            ] as $tax => $col){
                $val = $row[$map[$col]] ?? '';
                if($val){
                    $terms = array_map('trim', explode(',', $val));
                    foreach($terms as $t){ if(!term_exists($t, $tax)){ wp_insert_term($t, $tax); } }
                    wp_set_post_terms($post_id, $terms, $tax, false);
                }
            }

            foreach([
                '_min_deposit' => 'min_deposit',
                '_rating' => 'rating',
                '_spread_from' => 'spread_from',
                '_leverage' => 'leverage',
                '_affiliate_url' => 'affiliate_url',
            ] as $meta_key => $col){
                $val = sanitize_text_field($row[$map[$col]] ?? '');
                update_post_meta($post_id, $meta_key, $val);
            }

            $logo = $row[$map['logo_url']] ?? '';
            if($logo){
                $att_id = media_sideload_image( esc_url($logo), $post_id, $name, 'id' );
                if(!is_wp_error($att_id)){ set_post_thumbnail($post_id, $att_id); }
            }
            $count++;
        }
        fclose($fh);
        wp_redirect( admin_url('edit.php?post_type=broker&imported='.$count) );
        exit;
    }

    /* === Admin list columns === */
    public function admin_columns($cols){
        $cols['min_deposit'] = __('Min Dep','forex-brokers-directory');
        $cols['rating'] = __('Rating','forex-brokers-directory');
        $cols['country'] = __('Country','forex-brokers-directory');
        $cols['shortlink'] = __('Shortlink','forex-brokers-directory');
        return $cols;
    }
    public function admin_columns_render($col, $post_id){
        if($col==='min_deposit') echo esc_html(get_post_meta($post_id,'_min_deposit',true));
        if($col==='rating') echo esc_html(get_post_meta($post_id,'_rating',true));
        if($col==='country'){
            $terms = wp_get_post_terms($post_id,'broker_country',['fields'=>'names']);
            echo esc_html($terms ? implode(', ', $terms) : '');
        }
        if($col==='shortlink'){
            $post = get_post($post_id);
            echo esc_html( home_url('/go/'.$post->post_name.'/') );
        }
    }

    /* === Front-end Output === */
    public function shortcode_directory($atts) {
        $opt = get_option($this->opt_key);
        $atts = shortcode_atts([
            'per_page' => intval($opt['per_page'] ?? 12),
            'country'  => '',
            'cta'      => $opt['cta_text'] ?? __('Visit Broker','forex-brokers-directory'),
            'sort'     => isset($_GET['fbdsort']) ? sanitize_text_field($_GET['fbdsort']) : 'title',
            'paged'    => isset($_GET['fbdpage']) ? max(1, intval($_GET['fbdpage'])) : 1,
        ], $atts, 'forex_brokers');

        $orderby = 'title'; $meta_key = ''; $order = 'ASC';
        if($atts['sort']==='rating'){ $orderby='meta_value_num'; $meta_key='_rating'; $order='DESC'; }
        if($atts['sort']==='min'){ $orderby='meta_value_num'; $meta_key='_min_deposit'; $order='ASC'; }

        $args = [
            'post_type' => 'broker',
            'posts_per_page' => intval($atts['per_page']),
            'paged' => intval($atts['paged']),
            'post_status' => 'publish',
            'orderby' => $orderby,
            'order' => $order,
        ];
        if($meta_key){ $args['meta_key'] = $meta_key; }
        if ( !empty($atts['country']) ) {
            $args['tax_query'] = [[
                'taxonomy' => 'broker_country',
                'field' => 'slug',
                'terms' => sanitize_title($atts['country']),
            ]];
        }

        $q = new WP_Query($args);

        ob_start(); ?>
        <div class="fbd-wrap" id="fbdApp" data-cta="<?php echo esc_attr($atts['cta']); ?>">
          <div class="fbd-head">
            <div>
                <div class="fbd-title"><?php _e('Top Forex Brokers Directory','forex-brokers-directory'); ?></div>
                <div class="fbd-sub"><?php _e('Filter by country, platform, account type, rating, or min deposit.','forex-brokers-directory'); ?></div>
            </div>
            <div class="fbd-pillbar" id="fbdAlpha"></div>
          </div>

          <?php $this->render_filters(); ?>

          <div class="fbd-grid" id="fbdGrid">
            <?php $this->loop_cards($q, $atts['cta']); ?>
          </div>

          <?php $this->render_pagination($q, $atts); ?>
          <?php if(($opt['show_credit'] ?? '1')==='1'): ?><div class="fbd-credit" style="margin-top:10px;font-size:.82rem;color:#64748b;"></div><?php endif; ?>
        </div>
        <?php
        return ob_get_clean();
    }

    private function render_filters(){
        ?>
        <div class="fbd-filters" id="fbdFilters">
            <div>
              <label for="fbd-search"><?php _e('Search by broker name','forex-brokers-directory'); ?></label>
              <input type="search" id="fbd-search" placeholder="<?php esc_attr_e("Type a name e.g. 'Exness', 'XM'",'forex-brokers-directory'); ?>">
            </div>
            <div>
              <label for="fbd-country"><?php _e('Country','forex-brokers-directory'); ?></label>
              <select id="fbd-country">
                <option value=""><?php _e('All Countries','forex-brokers-directory'); ?></option>
                <?php $this->render_terms_options('broker_country'); ?>
              </select>
            </div>
            <div>
              <label for="fbd-platform"><?php _e('Platform','forex-brokers-directory'); ?></label>
              <select id="fbd-platform">
                <option value=""><?php _e('All Platforms','forex-brokers-directory'); ?></option>
                <?php $this->render_terms_options('broker_platform'); ?>
              </select>
            </div>
            <div>
              <label for="fbd-acct"><?php _e('Account Type','forex-brokers-directory'); ?></label>
              <select id="fbd-acct">
                <option value=""><?php _e('All Types','forex-brokers-directory'); ?></option>
                <?php $this->render_terms_options('broker_account'); ?>
              </select>
            </div>
            <div>
              <label><?php _e('Min Deposit (≤)','forex-brokers-directory'); ?></label>
              <div class="fbd-range">
                <input type="range" id="fbd-mindep" min="0" max="500" value="500" step="10" oninput="document.getElementById('fbd-mindepVal').value = this.value">
                <input type="number" id="fbd-mindepVal" min="0" max="500" value="500" step="10">
              </div>
            </div>
        </div>
        <?php
    }

    private function loop_cards($q, $cta_text){
        if ($q->have_posts()): while($q->have_posts()): $q->the_post();
            $id = get_the_ID();
            $name = get_the_title();
            $alpha = strtoupper(substr($name,0,1));
            $country_terms = wp_get_post_terms($id, 'broker_country', ['fields'=>'names']);
            $plat_terms = wp_get_post_terms($id, 'broker_platform', ['fields'=>'names']);
            $acct_terms = wp_get_post_terms($id, 'broker_account', ['fields'=>'names']);
            $country = $country_terms ? implode(',', $country_terms) : '';
            $platforms = $plat_terms ? implode(',', $plat_terms) : '';
            $accounts = $acct_terms ? implode(',', $acct_terms) : '';
            $min = get_post_meta($id, '_min_deposit', true);
            $rating = get_post_meta($id, '_rating', true);
            $spread = get_post_meta($id, '_spread_from', true);
            $lev = get_post_meta($id, '_leverage', true);
            $url = get_post_meta($id, '_affiliate_url', true);
            $logo = get_the_post_thumbnail_url($id, 'medium');
            if(!$logo){ $logo = $this->plugin_url('assets/placeholder-logo.png'); }
            $short = home_url('/go/'.get_post_field('post_name', $id).'/');
            ?>
            <article class="fbd-card"
              data-name="<?php echo esc_attr($name); ?>"
              data-country="<?php echo esc_attr($country); ?>"
              data-platforms="<?php echo esc_attr($platforms); ?>"
              data-accounts="<?php echo esc_attr($accounts); ?>"
              data-min="<?php echo esc_attr($min ? $min : 0); ?>"
              data-rating="<?php echo esc_attr($rating ? $rating : 0); ?>"
              data-alpha="<?php echo esc_attr($alpha); ?>">
              <div class="fbd-card-top">
                <div class="fbd-logo-wrap">
                    <img loading="lazy" class="fbd-logo" src="<?php echo esc_url($logo); ?>" alt="<?php echo esc_attr($name); ?>">
                </div>
                <div class="fbd-card-head">
                  <div class="fbd-name"><?php echo esc_html($name); ?></div>
                  <?php if($rating!==''): ?>
                    <div class="fbd-stars" aria-label="<?php echo esc_attr($rating); ?>">
                      <?php
                        $full = floor(floatval($rating));
                        $half = (floatval($rating) - $full) >= 0.5 ? 1 : 0;
                        $empty = 5 - $full - $half;
                        echo str_repeat('<span class="fbd-star full">★</span>', $full);
                        echo $half ? '<span class="fbd-star half">★</span>' : '';
                        echo str_repeat('<span class="fbd-star empty">☆</span>', $empty);
                      ?>
                      <span class="fbd-rating-num">(<?php echo esc_html($rating); ?>)</span>
                    </div>
                  <?php endif; ?>
                </div>
              </div>

              <div class="fbd-meta">
                <?php if($country): ?><span><strong><?php _e('Regulation:','forex-brokers-directory'); ?></strong> <?php echo esc_html( get_post_meta($id,'_regulation', true) ?: '-' ); ?></span><?php endif; ?>
                <?php if($country): ?><span><strong><?php _e('Country:','forex-brokers-directory'); ?></strong> <?php echo esc_html($country); ?></span><?php endif; ?>
                <?php if($platforms): ?><span><strong><?php _e('Platforms:','forex-brokers-directory'); ?></strong> <?php echo esc_html($platforms); ?></span><?php endif; ?>
                <?php if($accounts): ?><span><strong><?php _e('Account Types:','forex-brokers-directory'); ?></strong> <?php echo esc_html($accounts); ?></span><?php endif; ?>
                <?php if($min!==''): ?><span><strong><?php _e('Min Deposit:','forex-brokers-directory'); ?></strong> $<?php echo esc_html($min); ?></span><?php endif; ?>
                <?php if($spread): ?><span><strong><?php _e('Spread From:','forex-brokers-directory'); ?></strong> <?php echo esc_html($spread); ?></span><?php endif; ?>
                <?php if($lev): ?><span><strong><?php _e('Leverage:','forex-brokers-directory'); ?></strong> <?php echo esc_html($lev); ?></span><?php endif; ?>
              </div>

              <div class="fbd-badges">
                <?php if($min!==''): ?><span class="fbd-badge"><?php _e('Min: $','forex-brokers-directory'); echo esc_html($min); ?></span><?php endif; ?>
                <?php if($spread): ?><span class="fbd-badge"><?php echo esc_html($spread); ?></span><?php endif; ?>
                <?php if($lev): ?><span class="fbd-badge"><?php echo esc_html($lev); ?></span><?php endif; ?>
              </div>

              <div class="fbd-cta">
                <?php if($url): ?>
                  <a class="fbd-btn fbd-btn-primary" href="<?php echo esc_url($short); ?>" rel="nofollow sponsored"><?php echo esc_html($cta_text); ?></a>
                <?php endif; ?>
                <span class="fbd-note"><?php _e('Advertiser Disclosure: Affiliate link.','forex-brokers-directory'); ?></span>
              </div>
            </article>
            <?php
        endwhile; wp_reset_postdata(); else: ?>
            <p><?php _e('No brokers found. Add some under Brokers → Add New.','forex-brokers-directory'); ?></p>
        <?php endif;
    }

    private function render_pagination($q, $atts){
        $links = paginate_links([
            'base' => add_query_arg('fbdpage','%#%'),
            'format' => '',
            'current' => max(1, $atts['paged']),
            'total' => $q->max_num_pages,
            'prev_text' => __('← Prev','forex-brokers-directory'),
            'next_text' => __('Next →','forex-brokers-directory'),
            'type' => 'array'
        ]);
        if($links){
            echo '<nav class="fbd-pagination">';
            foreach($links as $l){ echo '<span class="fbd-page">'.$l.'</span>'; }
            echo '</nav>';
        }
    }

    private function render_terms_options($taxonomy){
        $terms = get_terms(['taxonomy'=>$taxonomy,'hide_empty'=>false]);
        if ( ! empty($terms) && ! is_wp_error($terms) ) {
            foreach ($terms as $t) {
                echo '<option value="'.esc_attr($t->name).'">'.esc_html($t->name).'</option>';
            }
        }
    }

    /* === Table variant === */
    public function shortcode_table($atts){
        $opt = get_option($this->opt_key);
        $atts = shortcode_atts([ 'per_page' => intval($opt['per_page'] ?? 12), 'paged' => isset($_GET['fbdpage']) ? max(1, intval($_GET['fbdpage'])) : 1 ], $atts, 'forex_brokers_table');

        $q = new WP_Query([
            'post_type' => 'broker',
            'posts_per_page' => intval($atts['per_page']),
            'paged' => intval($atts['paged']),
            'post_status' => 'publish',
            'orderby' => 'title',
            'order' => 'ASC',
        ]);

        ob_start(); ?>
        <div class="fbd-wrap">
          <div class="fbd-title"><?php _e('Forex Brokers Comparison Table','forex-brokers-directory'); ?></div>
          <div class="fbd-table-scroll">
            <table class="fbd-table">
              <thead>
                <tr>
                  <th><?php _e('Logo','forex-brokers-directory'); ?></th>
                  <th><?php _e('Broker','forex-brokers-directory'); ?></th>
                  <th><?php _e('Country','forex-brokers-directory'); ?></th>
                  <th><?php _e('Platforms','forex-brokers-directory'); ?></th>
                  <th><?php _e('Min Dep','forex-brokers-directory'); ?></th>
                  <th><?php _e('Spread From','forex-brokers-directory'); ?></th>
                  <th><?php _e('Leverage','forex-brokers-directory'); ?></th>
                  <th><?php _e('Action','forex-brokers-directory'); ?></th>
                </tr>
              </thead>
              <tbody>
                <?php if($q->have_posts()): while($q->have_posts()): $q->the_post();
                    $id = get_the_ID();
                    $name = get_the_title();
                    $logo = get_the_post_thumbnail_url($id, 'thumbnail');
                    if(!$logo){ $logo = $this->plugin_url('assets/placeholder-logo.png'); }
                    $country = wp_get_post_terms($id, 'broker_country', ['fields'=>'names']);
                    $platforms = wp_get_post_terms($id, 'broker_platform', ['fields'=>'names']);
                    $min = get_post_meta($id, '_min_deposit', true);
                    $spread = get_post_meta($id, '_spread_from', true);
                    $lev = get_post_meta($id, '_leverage', true);
                    $url = home_url('/go/'.get_post_field('post_name',$id).'/');
                ?>
                <tr>
                  <td><img loading="lazy" class="fbd-logo fbd-logo--table" src="<?php echo esc_url($logo); ?>" alt="<?php echo esc_attr($name); ?>"></td>
                  <td><?php echo esc_html($name); ?></td>
                  <td><?php echo esc_html($country ? implode(', ', $country) : ''); ?></td>
                  <td><?php echo esc_html($platforms ? implode(', ', $platforms) : ''); ?></td>
                  <td><?php echo $min!=='' ? '$'.esc_html($min) : ''; ?></td>
                  <td><?php echo esc_html($spread); ?></td>
                  <td><?php echo esc_html($lev); ?></td>
                  <td><a class="fbd-btn fbd-btn-primary" href="<?php echo esc_url($url); ?>" rel="nofollow sponsored"><?php echo esc_html($opt['cta_text'] ?? 'Visit Broker'); ?></a></td>
                </tr>
                <?php endwhile; wp_reset_postdata(); else: ?>
                <tr><td colspan="8"><?php _e('No brokers found.','forex-brokers-directory'); ?></td></tr>
                <?php endif; ?>
              </tbody>
            </table>
          </div>
          <?php $this->render_pagination($q, $atts); ?>
        </div>
        <?php
        return ob_get_clean();
    }
}

new Forex_Brokers_Directory();
''')

css = textwrap.dedent(r'''/* v1.2.0 - screenshot-style UI, mobile-first */
:root{
  --gap:16px; --radius:14px; --border:#e6e8ec; --text:#0b1220; --muted:#6b7280; --bg:#ffffff;
  --cta:#111827; --cta-text:#ffffff; --cta-hover:#0b1220;
}
.fbd-wrap{max-width:1200px;margin:0 auto;padding:20px;background:var(--bg);color:var(--text);}
.fbd-head{display:flex;flex-wrap:wrap;gap:var(--gap);align-items:center;justify-content:space-between;margin-bottom:18px;}
.fbd-title{font-size:1.6rem;font-weight:800;}
.fbd-sub{font-size:.95rem;color:var(--muted);}

.fbd-pillbar{display:flex;gap:8px;align-items:center}
.fbd-pillbar .fbd-pill{border:1px solid var(--border);padding:6px 10px;border-radius:999px;background:#fff;cursor:pointer;font-size:.85rem}
.fbd-pillbar .fbd-pill.active{background:#111827;color:#fff;border-color:#111827}

.fbd-filters{display:grid;gap:12px;grid-template-columns:repeat(5,1fr);align-items:end;margin:14px 0 18px;}
.fbd-filters label{font-size:.82rem;font-weight:800;margin-bottom:6px;display:block;}
.fbd-filters input[type="search"], .fbd-filters select, .fbd-filters input[type="number"], .fbd-filters input[type="range"]{
  width:100%;padding:12px;border:1px solid var(--border);border-radius:12px;font-size:.95rem;background:#fff;
}
.fbd-range{display:flex;align-items:center;gap:10px}
/* prettier range */
.fbd-filters input[type="range"]{appearance:none;height:6px;background:#e5e7eb;border-radius:4px}
.fbd-filters input[type="range"]::-webkit-slider-thumb{appearance:none;width:18px;height:18px;border-radius:50%;background:#111827;border:2px solid #fff;box-shadow:0 0 0 1px #111827;cursor:pointer}
.fbd-filters input[type="range"]::-moz-range-thumb{width:18px;height:18px;border-radius:50%;background:#111827;border:2px solid #fff}
@media (max-width:1024px){ .fbd-filters{grid-template-columns:repeat(3,1fr);} }
@media (max-width:680px){ .fbd-filters{grid-template-columns:1fr 1fr;} }

/* Card grid */
.fbd-grid{display:grid;gap:var(--gap);grid-template-columns:repeat(3,1fr);}
@media (max-width:1100px){ .fbd-grid{grid-template-columns:repeat(2,1fr);} }
@media (max-width:640px){ .fbd-grid{grid-template-columns:1fr;} }

.fbd-card{border:1px solid var(--border);border-radius:16px;padding:18px;display:flex;flex-direction:column;gap:14px;background:#fff;box-shadow:0 3px 14px rgba(2,8,23,.06);}
.fbd-card-top{display:flex;align-items:center;gap:14px;}
.fbd-logo-wrap{flex:0 0 auto; display:flex; align-items:center; justify-content:center; width:120px;}
.fbd-logo{width:120px;max-width:100%;height:auto;object-fit:contain}
.fbd-card-head{display:flex;flex-direction:column;gap:4px}
.fbd-name{font-weight:800;font-size:1.05rem;}
.fbd-stars{display:flex;align-items:center;gap:6px;color:#111827;font-size:.95rem;font-weight:700}
.fbd-stars .full,.fbd-stars .half{color:#111827}
.fbd-stars .empty{color:#d1d5db}
.fbd-rating-num{color:#111827;margin-left:2px}

.fbd-meta{font-size:.95rem;line-height:1.5}
.fbd-meta span{display:block;}
.fbd-badges{display:flex;flex-wrap:wrap;gap:8px}
.fbd-badge{font-size:.78rem;border:1px solid var(--border);padding:6px 10px;border-radius:999px;color:#0f172a;background:#f8fafc}

.fbd-cta{display:flex;gap:12px;align-items:center;justify-content:flex-start;margin-top:6px}
.fbd-btn{display:inline-block;padding:12px 16px;border-radius:12px;text-decoration:none;font-weight:800;border:1px solid transparent;transition:all .15s;}
.fbd-btn-primary{background:var(--cta);color:var(--cta-text);}
.fbd-btn-primary:hover{background:var(--cta-hover);transform:translateY(-1px)}
.fbd-note{font-size:.8rem;color:#9ca3af}

.fbd-hide{display:none!important}

/* Table variant */
.fbd-table-scroll{ overflow-x:auto; -webkit-overflow-scrolling:touch; }
.fbd-table{ width:100%; border-collapse:collapse; font-size:.96rem; }
.fbd-table th, .fbd-table td{ border:1px solid var(--border); padding:12px; text-align:left; white-space:nowrap; }
.fbd-logo--table{ width:120px; height:auto; object-fit:contain; }
@media (max-width: 680px){
  .fbd-logo--table{ width:90px; }
  .fbd-table th:nth-child(3), .fbd-table td:nth-child(3),
  .fbd-table th:nth-child(6), .fbd-table td:nth-child(6){ display:none; }
}

/* Pagination */
.fbd-pagination{display:flex;gap:8px;justify-content:center;margin-top:16px}
.fbd-page a, .fbd-page span{display:inline-block;padding:8px 12px;border:1px solid var(--border);border-radius:10px;text-decoration:none}
.fbd-page .current{background:#111827;color:#fff;border-color:#111827}
''')

js = textwrap.dedent(r'''(function(){
  function $(s,c=document){ return c.querySelector(s); }
  function $$(s,c=document){ return Array.from(c.querySelectorAll(s)); }

  document.addEventListener('DOMContentLoaded', function(){
    var app = $('#fbdApp');
    if(!app) return;

    var grid = $('#fbdGrid');
    var search = $('#fbd-search');
    var country = $('#fbd-country');
    var platform = $('#fbd-platform');
    var acct = $('#fbd-acct');
    var mindep = $('#fbd-mindep');
    var mindepVal = $('#fbd-mindepVal');
    var alphaBar = $('#fbdAlpha');

    // Build A–Z pills based on existing cards
    if(alphaBar){
      var lettersSet = new Set($$('.fbd-card', grid).map(function(c){ 
        var a = (c.dataset.alpha || c.dataset.name?.charAt(0) || '').toUpperCase(); 
        return a || '#';
      }));
      var letters = Array.from(lettersSet).sort();
      var pillAll = document.createElement('button');
      pillAll.className = 'fbd-pill active'; pillAll.textContent = 'All'; pillAll.dataset.alpha = '';
      alphaBar.appendChild(pillAll);
      letters.forEach(function(L){
        var b = document.createElement('button');
        b.className = 'fbd-pill'; b.textContent = L; b.dataset.alpha = L;
        alphaBar.appendChild(b);
      });
    }

    var alphaFilter = '';
    alphaBar && alphaBar.addEventListener('click', function(e){
      var btn = e.target.closest('.fbd-pill'); if(!btn) return;
      $$('.fbd-pill', alphaBar).forEach(function(p){ p.classList.remove('active'); });
      btn.classList.add('active');
      alphaFilter = btn.dataset.alpha || '';
      filter();
    });

    // Sync range <-> number
    if(mindep && mindepVal){
      mindepVal.addEventListener('input', function(){ mindep.value = mindepVal.value; filter(); });
      mindep.addEventListener('input', function(){ mindepVal.value = mindep.value; filter(); });
    }

    // Attach filter events
    [search, country, platform, acct].forEach(function(el){ if(el) el.addEventListener('input', filter); });

    function includesCSV(needle, csv){
      if(!needle) return true;
      return csv.toLowerCase().split(',').map(function(s){return s.trim();}).includes(needle.toLowerCase());
    }

    function filter(){
      var q = (search && search.value || '').trim().toLowerCase();
      var ctry = country && country.value || '';
      var plat = platform && platform.value || '';
      var acc  = acct && acct.value || '';
      var maxMin = parseFloat(mindep && mindep.value || '999999');

      $$('.fbd-card', grid).forEach(function(card){
        var name = (card.dataset.name || '').toLowerCase();
        var alpha = (card.dataset.alpha || (name[0]||'')).toUpperCase();
        var cardC = card.dataset.country || '';
        var cardP = card.dataset.platforms || '';
        var cardA = card.dataset.accounts || '';
        var min   = parseFloat(card.dataset.min || '0');

        var okSearch = !q || name.includes(q);
        var okCountry = !ctry || cardC.split(',').map(function(x){return x.trim();}).includes(ctry);
        var okPlat = includesCSV(plat, cardP);
        var okAcct = includesCSV(acc, cardA);
        var okMin = isNaN(maxMin) ? true : (min <= maxMin);
        var okAlpha = !alphaFilter || alpha === alphaFilter;

        var show = okSearch && okCountry && okPlat && okAcct && okMin && okAlpha;
        card.classList.toggle('fbd-hide', !show);
      });
    }

    filter(); // init
  });
})();''')

placeholder_logo = b'\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00P\x00\x00\x00\x18\x08\x06\x00\x00\x00\xd5\xe5\x0f\xc4\x00\x00\x00\x19tEXtSoftware\x00Adobe ImageReadyq\xc9e<\x00\x00\x00RIDATx\xdat\x91\xb1\x0e\xc0 \x08D\xd3\xff?\xa3\x0e;\x90\xb8\x9b\x85\xaa\x1d\xbd\x80\x80\x98\xa7\x9c9\xe0\xb9\xd3\x06\x9a\x08\x92\xd6\xa1\xaa\xd5\x9a\xf3|\x9f\x16\xbf\x07\xe0\xd1\x1e\xa1\x0c\xc3\x10\xc7q\x9c\xe7\xf9\x9e\x00\xf6\x1dI2\x99\x0c\x00\x00\x00\x00IEND\xaeB`\x82'

open(os.path.join(base, "forex-brokers-directory.php"), "w", encoding="utf-8").write(php)
open(os.path.join(assets, "style.css"), "w", encoding="utf-8").write(css)
open(os.path.join(assets, "script.js"), "w", encoding="utf-8").write(js)
open(os.path.join(assets, "placeholder-logo.png"), "wb").write(placeholder_logo)

zip_path = "/mnt/data/forex-brokers-directory-v1.2.0.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for root, dirs, files in os.walk(base):
        for name in files:
            full = os.path.join(root, name)
            arc = os.path.relpath(full, os.path.dirname(base))
            z.write(full, arc)

zip_path
