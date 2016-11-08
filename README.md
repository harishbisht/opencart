# opencart plugin

##### Step 1: First go to the root directory of the project
##### Step 2: Then go to admin->view->template->sale->order_info.tpl

##### Step 3: Open the order_info.tpl and add the below code in fourth div

```
<a href="<?php echo $pickrr; ?>" target="_blank" data-toggle="tooltip" title="Pickrr" class="btn btn-success"><i class="fa fa-shopping-cart"></i></a>

```
##### Step 4: Now create a file pickrr.tpl in admin->view->template->sale folder and paste below code and save it

```
Order is placed at pickrr and your tracking id is  : <?php echo $tracking_id; ?>
<br><br>
Your manifest link is :  
<a target="_blank" href="<?php echo $manifest_link; ?>"> <?php echo $manifest_link; ?> </a>
<br><br>
Your tracking url is : 
<a target="_blank" href="http://pickrr.com/tracking/#?tracking_id=<?php echo $tracking_id; ?>"> http://pickrr.com/tracking/#?tracking_id=<?php echo $tracking_id; ?> </a>
```


##### Step 5: Now go to admin->controller->sale and open order.php and find the function public function info() and inside that function in line 894 paste the below code and save it

```
$data['pickrr'] = $this->url->link('sale/order/pickrr', 'token=' . $this->session->data['token'] . '&order_id=' . (int)$this->request->get['order_id'], true);
```

##### Step 6: Now again go to admin->controller->sale and open order.php and before the last curly braces add the below code and save it.


```
	public function pickrr(){
		$this->load->language('sale/order');
		$this->load->model('sale/order');
		$this->load->model('catalog/product');
		$this->load->model('setting/setting');
		$data['orders'] = array();

		if (isset($this->request->post['selected'])) {
			$orders = $this->request->post['selected'];
		} elseif (isset($this->request->get['order_id'])) {
			$orders[] = $this->request->get['order_id'];
		}
		// fetching destination address
		foreach ($orders as $order_id) {
			$order_info = $this->model_sale_order->getOrder($order_id);
			// Make sure there is a shipping method
			if ($order_info && $order_info['shipping_code']) {
				$data['to_name'] = $order_info['shipping_firstname']. " ".$order_info['shipping_lastname'];
				$data['to_phone_number'] = $order_info['telephone'];
				$data['to_pincode'] = $order_info['shipping_postcode'];
				$data['to_address'] = $order_info['shipping_address_1']." ".$order_info['shipping_address_2']." ".$order_info['shipping_company']." ".$order_info['shipping_city']." ".$order_info['shipping_zone']." ".$order_info['shipping_zone_code']." ".$order_info['shipping_country'];
				$data['client_order_id'] = $order_id;
			}
		}

		$data['cod_amount'] = 0;
		$data['item_name'] = "";

		// fetching itemname and payment details
		$this->load->language('sale/order');
		foreach ($orders as $order_id) {
			$order_info = $this->model_sale_order->getOrder($order_id);
			if ($order_info) {
				$this->load->model('tool/upload');
				$product_data = array();
				$products = $this->model_sale_order->getOrderProducts($order_id);
				foreach ($products as $product) {
					$data['item_name'] =$data['item_name'].", ".$product['name'];
				}
				$data['payment_code'] = $order_info['payment_code'];
			}
		}

		$totals = $this->model_sale_order->getOrderTotals($order_id);

		// checking order is cod or prepaid
		if ($data['payment_code'] == "cod") {
		    $data['cod_amount'] =end($totals)['value'];
		}
        elseif ($data['payment_code'] == "GOP_COD") {
            $data['cod_amount'] =end($totals)['value'];
        }
        elseif($data['payment_code'] == "payu") {
                $data['cod_amount'] =0.0;
            }
		else{
				$data['cod_amount'] =0.0;
			}
		$data['invoice_value'] = end($totals)['value'];
		$data['auth_token'] = "<YOUR-AUTH-TOKEN>";
		$data['from_name'] = "<YOUR-NAME>";
		$data['from_phone_number'] = "<YOUR-PHONE-NUMBER>";
		$data['from_pincode'] = "<YOUR-PINCODE>";
		$data['from_address'] = "<YOUR-ADDRESS>";


	    try{
          $params = array(
                      'auth_token' => $data['auth_token'],
                      'item_name' => $data['item_name'],
                      'from_name' => $data['from_name'],
                      'from_phone_number' => $data['from_phone_number'],
                      'from_pincode'=> $data['from_pincode'],
                      'from_address'=> $data['from_address'],
                      'to_name'=> $data['to_name'],
                      'to_phone_number' => $data['to_phone_number'],
                      'to_pincode' => $data['to_pincode'],
                      'to_address' => $data['to_address'],
                      'client_order_id' => $data['client_order_id'],
                      'cod_amount' => $data['cod_amount'],
                      'invoice_value' => $data['invoice_value']
                    );
            $json_params = json_encode( $params );
            $url = 'http://www.pickrr.com/api/place-order/';
            //open connection
            $ch = curl_init();
            //set the url, number of POST vars, POST data
            curl_setopt($ch,CURLOPT_URL, $url);
            curl_setopt($ch,CURLOPT_POSTFIELDS, $json_params);
            curl_setopt( $ch, CURLOPT_HTTPHEADER, array('Content-Type:application/json'));
            curl_setopt( $ch, CURLOPT_RETURNTRANSFER, true );
            //execute post
            $result = curl_exec($ch);
            $result = json_decode($result, true);
            //close connection
            curl_close($ch);
            if(gettype($result)!="array")
              throw new \Exception( print_r($result, true) . "Problem in connecting with Pickrr");
            if($result['err']!="")
              throw new \Exception($result['err']);
            $data['tracking_id'] = $result['tracking_id'];
            $data['manifest_link'] = $result['manifest_link'];

            // adding shipping details
			$tracking = "Your tracking id is : ".$result['tracking_id'];
			$queryset = 'insert into <DATABASE-NAME>.oc_order_history (order_id,order_status_id,notify,comment,date_added) values(orderid,3,0,"tracking",NOW());';
			$queryset = str_replace('tracking', $tracking, $queryset);
			$queryset = str_replace('orderid', $this->request->get['order_id'], $queryset);
			$sql = $this->db->query($queryset);

			$tracking = "http://pickrr.com/tracking/#?tracking_id=".$result['tracking_id'];
			$queryset = 'insert into <DATABASE-NAME>.oc_order_history (order_id,order_status_id,notify,comment,date_added) values(orderid,3,0,"tracking",NOW());';
			$queryset = str_replace('tracking', $tracking, $queryset);
			$queryset = str_replace('orderid', $this->request->get['order_id'], $queryset);
			$sql = $this->db->query($queryset);

			$tracking = "Your manifest link is : ".$result['manifest_link'];
			$queryset = 'insert into <DATABASE-NAME>.oc_order_history (order_id,order_status_id,notify,comment,date_added) values(orderid,3,0,"tracking",NOW());';
			$queryset = str_replace('tracking', $tracking, $queryset);
			$queryset = str_replace('orderid', $this->request->get['order_id'], $queryset);
			$sql = $this->db->query($queryset);

        }
        catch (\Exception $e) {
            $data['tracking_id']= $e->getMessage();
        }
	$this->response->setOutput($this->load->view('sale/pickrr', $data));
	}

```
##### Step 7: In above code please add your auth-token, name, pincode,phone number and address and change database name, save all the changes and now go to your order and click on pickrr button for placing order in pickrr

##### Step 8: For any help please email us to harish@pickrr.com or info@pickrr.com
