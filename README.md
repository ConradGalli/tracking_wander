# tracking_wander.

<?php
/*
	Plugin Name: WooCommerce Shipment Tracking
	Plugin URI: http://shop.wanderlust-webdesign.com
	Description: Add tracking numbers to orders allowing customers to track their orders via a link. Supports many shipping providers, as well as custom ones if neccessary via a regular link.
	Version: 5.3
	Author: shop.wanderlust-webdesign.com
	Author URI: http://shop.wanderlust-webdesign.com

	Copyright: © 2009-2012 shop.wanderlust-webdesign.com.
	License: GNU General Public License v3.0
	License URI: http://www.gnu.org/licenses/gpl-3.0.html
*/

/**
 * Required functions
 */


if ( is_woocommerce_active() ) {

	/**
	 * WC_Shipment_Tracking class
	 */
	if ( ! class_exists( 'WC_Shipment_Tracking' ) ) {

		class WC_Shipment_Tracking {

			/**
			 * Constructor
			 */
			public function __construct() {
				add_action( 'admin_print_styles', array( $this, 'admin_styles' ) );
				add_action( 'add_meta_boxes', array( $this, 'add_meta_box' ) );
				add_action( 'woocommerce_process_shop_order_meta', array( $this, 'save_meta_box' ), 0, 2 );
				add_action( 'plugins_loaded', array( $this, 'load_plugin_textdomain' ) );

				// View Order Page
				add_action( 'woocommerce_view_order', array( $this, 'display_tracking_info' ) );
				add_action( 'woocommerce_email_before_order_table', array( $this, 'email_display' ) );

				// Customer / Order CSV Export column headers/data
				add_filter( 'wc_customer_order_csv_export_order_headers',   array( $this, 'add_tracking_info_to_csv_export_column_headers' ) );
				add_filter( 'wc_customer_order_csv_export_order_row',       array( $this, 'add_tracking_info_to_csv_export_column_data' ), 10, 3 );

				// Prevent data being copied to subscriptions
				add_filter( 'woocommerce_subscriptions_renewal_order_meta_query', array( $this, 'woocommerce_subscriptions_renewal_order_meta_query' ), 10, 4 );
			}

			/**
			 * Get shiping providers
			 * @return array
			 */
			public function get_providers() {
				return apply_filters( 'wc_shipment_tracking_get_providers', array(
					'Australia' => array(
						'Australia Post'
							=> 'http://auspost.com.au/track/track.html?id=%1$s',
					),
					'Austria' => array(
						'post.at' =>
							'http://www.post.at/sendungsverfolgung.php?pnum1=%1$s',
						'dhl.at' =>
							'http://www.dhl.at/content/at/de/express/sendungsverfolgung.html?brand=DHL&AWB=%1$s'
					),
					'Brazil' => array(
						'Correios'
							=> 'http://websro.correios.com.br/sro_bin/txect01$.QueryList?P_LINGUA=001&P_TIPO=001&P_COD_UNI=%1$s'
					),
					'Canada' => array(
						'Canada Post'
							=> 'http://www.canadapost.ca/cpotools/apps/track/personal/findByTrackNumber?trackingNumber=%1$s',
					),
					'Czech Republic' => array(
						'PPL.cz'
							=> 'http://www.ppl.cz/main2.aspx?cls=Package&idSearch=%1$s',
						'Česká pošta'
							=> 'http://www.ceskaposta.cz/cz/nastroje/sledovani-zasilky.php?barcode=%1$s&locale=CZ&send.x=52&send.y=8&go=ok',
						'DHL.cz'
							=> 'http://www.dhl.cz/content/cz/cs/express/sledovani_zasilek.shtml?brand=DHL&AWB=%1$s',
						'DPD.cz'
							=> 'https://tracking.dpd.de/cgi-bin/delistrack?pknr=%1$s&typ=32&lang=cz',
					),
					'Finland' => array(
						'Itella'
							=> 'http://www.posti.fi/itemtracking/posti/search_by_shipment_id?lang=en&ShipmentId=%1$s',
					),
					'France' => array(
						'Colissimo'
							=> 'http://www.colissimo.fr/portail_colissimo/suivre.do?language=fr_FR&colispart=%1$s',
					),
					'Germany' => array(
						'DHL Intraship (DE)'
							=> 'http://nolp.dhl.de/nextt-online-public/set_identcodes.do?lang=de&idc=%1$s&rfn=&extendedSearch=true',
						'Hermes'
							=> 'https://tracking.hermesworld.com/?TrackID=%1$s',
						'Deutsche Post DHL'
							=> 'http://nolp.dhl.de/nextt-online-public/set_identcodes.do?lang=de&idc=%1$s',
						'UPS Germany'
							=> 'http://wwwapps.ups.com/WebTracking/processInputRequest?sort_by=status&tracknums_displayed=1&TypeOfInquiryNumber=T&loc=de_DE&InquiryNumber1=%1$s',
						'DPD'
							=> 'https://tracking.dpd.de/parcelstatus?query=%1$s&locale=en_DE',
					),
					'Ireland' => array(
						'DPD'
							=> 'http://www2.dpd.ie/Services/QuickTrack/tabid/222/ConsignmentID/%1$s/Default.aspx',
					),
					'Italy' => array(
						'BRT (Bartolini)'
							=> 'http://as777.brt.it/vas/sped_det_show.hsm?referer=sped_numspe_par.htm&Nspediz=%1$s',
						'DHL Express'
							=> 'http://www.dhl.it/it/express/ricerca.html?AWB=%1$s&brand=DHL'
					),
					'India' => array(
						'DTDC'
							=> 'http://www.dtdc.in/dtdcTrack/Tracking/consignInfo.asp?strCnno=%1$s',
					),
					'Netherlands' => array(
						'PostNL'
							=> 'https://mijnpakket.postnl.nl/Claim?Barcode=%1$s&Postalcode=%2$s&Foreign=False&ShowAnonymousLayover=False&CustomerServiceClaim=False',
						'DPD.NL'
							=> 'http://track.dpdnl.nl/?parcelnumber=%1$s',
					),
					'New Zealand' => array(
						'Courier Post'
							=> 'http://trackandtrace.courierpost.co.nz/Search/%1$s',
						'NZ Post'
							=> 'http://www.nzpost.co.nz/tools/tracking?trackid=%1$s',
						'Fastways'
							=> 'http://www.fastway.co.nz/courier-services/track-your-parcel?l=%1$s',
						'PBT Couriers'
							=> 'http://www.pbt.com/nick/results.cfm?ticketNo=%1$s',
					),
					'South African' => array(
						'SAPO'
							=> 'http://sms.postoffice.co.za/TrackingParcels/Parcel.aspx?id=%1$s',
					),
					'Sweden' => array(
						'Posten AB'
							=> 'http://www.posten.se/sv/Kundservice/Sidor/Sok-brev-paket.aspx?search=%1$s',
						'DHL.se'
							=> 'http://www.dhl.se/content/se/sv/express/godssoekning.shtml?brand=DHL&AWB=%1$s',
						'Bring.se'
							=> 'http://tracking.bring.se/tracking.html?q=%1$s',
						'UPS.se'
							=> 'http://wwwapps.ups.com/WebTracking/track?track=yes&loc=sv_SE&trackNums=%1$s',
						'DB Schenker'
							=> 'http://privpakportal.schenker.nu/TrackAndTrace/packagesearch.aspx?packageId=%1$s'
					),
					'United Kingdom' => array(
						'DHL'
							=> 'http://www.dhl.com/content/g0/en/express/tracking.shtml?brand=DHL&AWB=%1$s',
						'DPD'
							=> 'http://www.dpd.co.uk/tracking/trackingSearch.do?search.searchType=0&search.parcelNumber=%1$s',
						'InterLink'
							=> 'http://www.interlinkexpress.com/apps/tracking/?help_type=%2Fapps%2Ftracking%2F%23results&reference=%1$s&postcode=%2$s#results',
						'ParcelForce'
							=> 'http://www.parcelforce.com/portal/pw/track?trackNumber=%1$s',
						'Royal Mail'
							=> 'https://www.royalmail.com/track-your-item/?trackNumber=%1$s',
						'TNT Express (consignment)'
							=> 'http://www.tnt.com/webtracker/tracking.do?requestType=GEN&searchType=CON&respLang=en&
respCountry=GENERIC&sourceID=1&sourceCountry=ww&cons=%1$s&navigation=1&g
enericSiteIdent=',
						'TNT Express (reference)'
							=> 'http://www.tnt.com/webtracker/tracking.do?requestType=GEN&searchType=REF&respLang=en&r
espCountry=GENERIC&sourceID=1&sourceCountry=ww&cons=%1$s&navigation=1&gen
ericSiteIdent=',
						'UK Mail'
							=> 'https://old.ukmail.com/ConsignmentStatus/ConsignmentSearchResults.aspx?SearchType=Reference&SearchString=%1$s',
					),
					'United States' => array(
						'Fedex'
							=> 'http://www.fedex.com/Tracking?action=track&tracknumbers=%1$s',
						'FedEx Sameday'
							=> 'https://www.fedexsameday.com/fdx_dotracking_ua.aspx?tracknum=%1$s',
						'OnTrac'
							=> 'http://www.ontrac.com/trackingdetail.asp?tracking=%1$s',
						'UPS'
							=> 'http://wwwapps.ups.com/WebTracking/track?track=yes&trackNums=%1$s',
						'USPS'
							=> 'https://tools.usps.com/go/TrackConfirmAction_input?qtc_tLabels1=%1$s',
					),
				) );
			}

			/**
			 * Localisation
			 */
			public function load_plugin_textdomain() {
				load_plugin_textdomain( 'wc_shipment_tracking', false, dirname( plugin_basename( __FILE__ ) ) . '/languages/' );
			}

			public function admin_styles() {
				wp_enqueue_style( 'shipment_tracking_styles', plugins_url( basename( dirname( __FILE__ ) ) ) . '/assets/css/admin.css' );
			}

			/**
			 * Add the meta box for shipment info on the order page
			 *
			 * @access public
			 */
			public function add_meta_box() {
				add_meta_box( 'woocommerce-shipment-tracking', __('Shipment Tracking', 'wc_shipment_tracking'), array( $this, 'meta_box' ), 'shop_order', 'side', 'high');
			}

			/**
			 * Show the meta box for shipment info on the order page
			 *
			 * @access public
			 */
			public function meta_box() {
				global $woocommerce, $post;

				// Providers
				echo '<p class="form-field tracking_provider_field"><label for="tracking_provider">' . __('Provider:', 'wc_shipment_tracking') . '</label><br/><select id="tracking_provider" name="tracking_provider" class="chosen_select" style="width:100%;">';

				echo '<option value="">' . __('Custom Provider', 'wc_shipment_tracking') . '</option>';

				$selected_provider = get_post_meta( $post->ID, '_tracking_provider', true );

				if ( ! $selected_provider )
					$selected_provider = sanitize_title( apply_filters( 'woocommerce_shipment_tracking_default_provider', '' ) );

				foreach ( $this->get_providers() as $provider_group => $providers ) {

					echo '<optgroup label="' . $provider_group . '">';

					foreach ( $providers as $provider => $url ) {

						echo '<option value="' . sanitize_title( $provider ) . '" ' . selected( sanitize_title( $provider ), $selected_provider, true ) . '>' . $provider . '</option>';

					}

					echo '</optgroup>';

				}

				echo '</select> ';

				woocommerce_wp_text_input( array(
					'id' 			=> 'custom_tracking_provider',
					'label' 		=> __('Provider Name:', 'wc_shipment_tracking'),
					'placeholder' 	=> '',
					'description' 	=> '',
					'value'			=> get_post_meta( $post->ID, '_custom_tracking_provider', true )
				) );

				woocommerce_wp_text_input( array(
					'id' 			=> 'tracking_number',
					'label' 		=> __('Tracking number:', 'wc_shipment_tracking'),
					'placeholder' 	=> '',
					'description' 	=> '',
					'value'			=> get_post_meta( $post->ID, '_tracking_number', true )
				) );

				woocommerce_wp_text_input( array(
					'id' 			=> 'custom_tracking_link',
					'label' 		=> __('Tracking link:', 'wc_shipment_tracking'),
					'placeholder' 	=> 'http://',
					'description' 	=> '',
					'value'			=> get_post_meta( $post->ID, '_custom_tracking_link', true )
				) );

				woocommerce_wp_text_input( array(
					'id' 			=> 'date_shipped',
					'label' 		=> __('Date shipped:', 'wc_shipment_tracking'),
					'placeholder' 	=> 'YYYY-MM-DD',
					'description' 	=> '',
					'class'			=> 'date-picker-field',
					'value'			=> ( $date = get_post_meta( $post->ID, '_date_shipped', true ) ) ? date( 'Y-m-d', $date ) : ''
				) );

				// Live preview
				echo '<p class="preview_tracking_link">' . __('Preview:', 'wc_shipment_tracking') . ' <a href="" target="_blank">' . __('Click here to track your shipment', 'wc_shipment_tracking') . '</a></p>';

				$provider_array = array();

				foreach ( $this->get_providers() as $providers ) {
					foreach ( $providers as $provider => $format ) {
						$provider_array[sanitize_title( $provider )] = urlencode( $format );
					}
				}

				$js = "
					jQuery('p.custom_tracking_link_field, p.custom_tracking_provider_field').hide();

					jQuery('input#custom_tracking_link, input#tracking_number, #tracking_provider').change(function(){

						var tracking = jQuery('input#tracking_number').val();
						var provider = jQuery('#tracking_provider').val();
						var providers = jQuery.parseJSON( '" . json_encode( $provider_array ) . "' );

						var postcode = jQuery('#_shipping_postcode').val();

						if ( ! postcode )
							postcode = jQuery('#_billing_postcode').val();

						postcode = encodeURIComponent( postcode );

						var link = '';

						if ( providers[ provider ] ) {
							link = providers[provider];
							link = link.replace( '%251%24s', tracking );
							link = link.replace( '%252%24s', postcode );
							link = decodeURIComponent( link );

							jQuery('p.custom_tracking_link_field, p.custom_tracking_provider_field').hide();
						} else {
							jQuery('p.custom_tracking_link_field, p.custom_tracking_provider_field').show();

							link = jQuery('input#custom_tracking_link').val();
						}

						if ( link ) {
							jQuery('p.preview_tracking_link a').attr('href', link);
							jQuery('p.preview_tracking_link').show();
						} else {
							jQuery('p.preview_tracking_link').hide();
						}

					}).change();
				";

				if ( function_exists( 'wc_enqueue_js' ) ) {
					wc_enqueue_js( $js );
				} else {
					$woocommerce->add_inline_js( $js );
				}
			}

			/**
			 * Order Downloads Save
			 *
			 * Function for processing and storing all order downloads.
			 */
			public function save_meta_box( $post_id, $post ) {
				if ( isset( $_POST['tracking_number'] ) ) {

					// Download data
					$tracking_provider        = wc_clean( $_POST['tracking_provider'] );
					$custom_tracking_provider = wc_clean( $_POST['custom_tracking_provider'] );
					$custom_tracking_link     = wc_clean( $_POST['custom_tracking_link'] );
					$tracking_number          = wc_clean( $_POST['tracking_number'] );
					$date_shipped             = wc_clean( strtotime( $_POST['date_shipped'] ) );

					// Update order data
					update_post_meta( $post_id, '_tracking_provider', $tracking_provider );
					update_post_meta( $post_id, '_custom_tracking_provider', $custom_tracking_provider );
					update_post_meta( $post_id, '_tracking_number', $tracking_number );
					update_post_meta( $post_id, '_custom_tracking_link', $custom_tracking_link );
					update_post_meta( $post_id, '_date_shipped', $date_shipped );
				}
			}

			/**
			 * Display Shipment info in the frontend (order view/tracking page).
			 *
			 * @access public
			 */
			public function display_tracking_info( $order_id, $for_email = false ) {

				$tracking_provider = get_post_meta( $order_id, '_tracking_provider', true );
				$tracking_number   = get_post_meta( $order_id, '_tracking_number', true );
				$date_shipped      = get_post_meta( $order_id, '_date_shipped', true );
				$postcode          = get_post_meta( $order_id, '_shipping_postcode', true );

				if ( ! $postcode )
					$postcode		= get_post_meta( $order_id, '_billing_postcode', true );

				if ( ! $tracking_number )
					return;

				if ( $date_shipped )
					$date_shipped = ' ' . sprintf( __( 'el %s', 'wc_shipment_tracking' ), date_i18n( __( 'l j F Y', 'wc_shipment_tracking'), $date_shipped ) );

				$tracking_link = '';

				if ( $tracking_provider ) {

					$link_format = '';

					foreach ( $this->get_providers() as $providers ) {
						foreach ( $providers as $provider => $format ) {
							if ( sanitize_title( $provider ) == $tracking_provider ) {
								$link_format = $format;
								$tracking_provider = $provider;
								break;
							}
						}
						if ( $link_format ) break;
					}

					if ( $link_format ) {
						$link = sprintf( $link_format, $tracking_number, urlencode( $postcode ) );
						if ( $for_email ) {
							$tracking_link = sprintf( __('Haga clic aquí para rastrear su envío', 'wc_shipment_tracking') . ': <a href="%s">%s</a>', $link, $link );
						} else {
							$tracking_link = sprintf( '<a href="%s">' . __('Haga clic aquí para rastrear su envío', 'wc_shipment_tracking') . '.</a>', $link, $link );
						}
					}

					$tracking_provider = ' ' . __('por', 'wc_shipment_tracking') . ' <strong>' . $tracking_provider . '</strong>';

					echo wpautop( sprintf( __('Su pedido fue enviado%s%s. El número de rastreo %s. %s', 'wc_shipment_tracking'), $date_shipped, $tracking_provider, $tracking_number, $tracking_link ) );

				} else {

					$custom_tracking_link     = get_post_meta( $order_id, '_custom_tracking_link', true );
					$custom_tracking_provider = get_post_meta( $order_id, '_custom_tracking_provider', true );

					if ( $custom_tracking_provider )
						$tracking_provider = ' ' . __('por', 'wc_shipment_tracking') . ' <strong>' . $custom_tracking_provider . '</strong>';
					else
						$tracking_provider = '';

					if ( $custom_tracking_link ) {
						$tracking_link = sprintf( '<a href="%s">' . __('Haga clic aquí para rastrear su envío', 'wc_shipment_tracking') . '.</a>', $custom_tracking_link . $tracking_number );
					} elseif ( strstr( $tracking_number, '<a' ) ) {
						$tracking_link = sprintf( '<a href="%s">%s.</a>', $tracking_number, $tracking_number );
					} else {
						$tracking_link = '';
					}

					echo wpautop( sprintf( __('Su pedido fue enviado%s%s. El número de rastreo %s. %s', 'wc_shipment_tracking'), $date_shipped, $tracking_provider, $tracking_number, $tracking_link ) );
				}

			}

			/**
			 * Display shipment info in customer emails.
			 *
			 * @access public
			 * @return void
			 */
			public function email_display( $order ) {
				$this->display_tracking_info( $order->id, true );
			}

			/**
			 * Adds support for Customer/Order CSV Export by adding appropraite column headers
			 *
			 * @param array $headers existing array of header key/names for the CSV export
			 * @return array
			 */
			public function add_tracking_info_to_csv_export_column_headers( $headers ) {

				$headers['tracking_provider']        = 'tracking_provider';
				$headers['custom_tracking_provider'] = 'custom_tracking_provider';
				$headers['tracking_number']          = 'tracking_number';
				$headers['custom_tracking_link']     = 'custom_tracking_link';
				$headers['date_shipped']             = 'date_shipped';

				return $headers;
			}

			/**
			 * Adds support for Customer/Order CSV Export by adding data for the column headers
			 *
			 * @param array $order_data generated order data matching the column keys in the header
			 * @param WC_Order $order order being exported
			 * @param \WC_CSV_Export_Generator $csv_generator instance
			 * @return array
			 */
			public function add_tracking_info_to_csv_export_column_data( $order_data, $order, $csv_generator ) {
				$tracking_provider        = get_post_meta( $order->id, '_tracking_provider', true );
				$custom_tracking_provider = get_post_meta( $order->id, '_custom_tracking_provider', true );
				$tracking_number          = get_post_meta( $order->id, '_tracking_number', true );
				$custom_tracking_link     = get_post_meta( $order->id, '_custom_tracking_link', true );
				$date_shipped             = get_post_meta( $order->id, '_date_shipped', true );

				$tracking_data = array(
					'tracking_provider'        => $tracking_provider,
					'custom_tracking_provider' => $custom_tracking_provider,
					'tracking_number'          => $tracking_number,
					'custom_tracking_link'     => $custom_tracking_link,
					'date_shipped'             => $date_shipped ? date_i18n( __( 'Y-m-d', 'wc_shipment_tracking' ), $date_shipped ) : '',
				);

				$new_order_data = array();

				if ( isset( $csv_generator->order_format ) && ( 'default_one_row_per_item' == $csv_generator->order_format || 'legacy_one_row_per_item' == $csv_generator->order_format ) ) {

					foreach ( $order_data as $data ) {
						$new_order_data[] = array_merge( (array) $data, $tracking_data );
					}

				} else {

					$new_order_data = array_merge( $order_data, $tracking_data );
				}

				return $new_order_data;
			}

			/**
			 * Prevents data being copied to subscription renewals
			 */
			public function woocommerce_subscriptions_renewal_order_meta_query( $order_meta_query, $original_order_id, $renewal_order_id, $new_order_role ) {
				$order_meta_query .= " AND `meta_key` NOT IN ( '_tracking_provider', '_custom_tracking_provider', '_tracking_number', '_custom_tracking_link', '_date_shipped' )";
				return $order_meta_query;
			}
		}
	}

	/**
	 * Register this class globally
	 */
	$GLOBALS['WC_Shipment_Tracking'] = new WC_Shipment_Tracking();
}
