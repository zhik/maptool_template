<script>
  import { getContext } from 'svelte'
  import { layers, info } from '../../stores'
  import { carto_apikey, geoclient_id, geoclient_key } from '../../utils/keys'
  1
  const { getMap, getCartoClient, getPopup } = getContext(carto_apikey)
  const map = getMap()
  const client = getCartoClient()
  const popup = getPopup()

  //query SLA data source from Carto and stack multiple entries at the same location
  const source = new carto.source.SQL(`
		WITH
			g AS (
				SELECT cartodb_id, ST_Transform(ST_GeomFromText(georeference, 4326),3857) AS the_geom_webmercator
				FROM liquor_authority_quarterly_list_of_active_licenses WHERE georeference != '' AND zone = 1 AND license_type_code in ('OP','SW','SB','SL','VL','RL','HL','CL','CT','EL','TL','CR','RW','HW','CW','TW','WC','EB','MR')
			),
			m AS (
				SELECT array_agg(cartodb_id) AS id_list, the_geom_webmercator, ST_Y(the_geom_webmercator) AS y
				FROM g
				GROUP BY the_geom_webmercator
				ORDER BY y DESC
			),
			f AS (
				SELECT generate_series(1, array_length(id_list,1)) AS p, unnest(id_list) AS cartodb_id, the_geom_webmercator
				FROM m
			)
      SELECT ST_Translate(f.the_geom_webmercator,0,f.p*5) AS the_geom_webmercator, q.cartodb_id, premise_name, dba, license_type_code, license_issued_date, license_expiration_date, serial_number, certificate_number, premise_address, premise_zip,
        method_of_operation, days_hours_of_operation,
				EXTRACT(year from to_date(license_expiration_date, 'MM/DD/YYYY')) AS expiration_year
				FROM f, liquor_authority_quarterly_list_of_active_licenses q
				WHERE f.cartodb_id = q.cartodb_id		
        `)

  //Style the SLA data and color differently if expiring this year
  const style = new carto.style.CartoCSS(`
        #layer {
            marker-width: 20;
            marker-fill-opacity: 0.9;
            marker-file: url('https://s3.amazonaws.com/com.cartodb.users-assets.production/maki-icons/bar-18.svg');
            marker-allow-overlap: true;
            marker-line-width: 1;
            marker-line-color: #ffffff;
            marker-line-opacity: 1;
        }
        #layer [expiration_year <=2019] {
            marker-fill: #e10012;
        }
        #layer [expiration_year > 2019] {
            marker-fill: #e34dee;
        }
        #layer [zoom > 16] {
            marker-width: 20;
        }
        #layer [zoom <=16] {
            marker-width: 15;
        }
        #layer [zoom <=15] {
            marker-width: 13;
        }
        #layer [zoom <=14] {
            marker-width: 8;
        }
        #layer [zoom <=12] {
            marker-width: 4;
        }
    `)

  const layer = new carto.layer.Layer(source, style, {
    featureClickColumns: [
      'premise_name',
      'dba', // doing_business_as_dba_
      'serial_number',
      'license_type_code', // license_type_name, TODO - lookup
      'license_issued_date', //license_original_issue_date does not exist, remove references; exist as license_effective_date
      'license_expiration_date',
      'certificate_number',
      'premise_address', //actual_address_of_premises_address1_, TODO - combine address part 2
      'premise_zip', //'zip'
      'method_of_operation',
      'days_hours_of_operation'
    ]
  })

  client
    .addLayer(layer)
    .then(() => {
      //add layer to state
      layers.add({
        order: 1,
        ref: layer,
        label: 'State Liquor Authority Licenses',
        legend: [
          {
            image:
              'https://s3.amazonaws.com/com.cartodb.users-assets.production/production/betanyc/assets/20180629205705bar-15.svg',
            text: 'Expired before January 1, 2020'
          },
          {
            image:
              'https://s3.amazonaws.com/com.cartodb.users-assets.production/production/betanyc/assets/20180629205836bar-15.svg',
            text: 'Expiring after January 1, 2020'
          }
        ],
        checked: true
      })
    })
    .catch(error => console.log(error.message))

  //set pop-up content when hovering over a feature
  layer.on('featureOver', featureEvent => {
    let address = ''

    address += `<div class="widget">`

    address += `<p class = "bold">${featureEvent.data.premise_address}</p><p>${featureEvent.data.dba}</p><p>${featureEvent.data.premise_name}</p>`
    popup.setContent(address)
    popup.setLatLng(featureEvent.latLng)
    if (!popup.isOpen()) {
      popup.openOn(map)
    }
  })

  layer.on('featureClicked', featureEvent => {
    //variables for the innerHTML content that will be filled into each div in the infobox
    let content = ''
    let source = ''

    //variable to store BIN from city geoclient
    var bin = 0

    //check if dba is filled in; otherwise use premises name as the header
    if (featureEvent.data.doing_business_as_dba != '') {
      content += `<h3 class = "bold">${featureEvent.data.dba}</h3><h4 class = "bold">${featureEvent.data.premise_name}</h4>`
    } else {
      content += `<h3 class = "bold">${featureEvent.data.premise_name}</h3>`
    }
    content += `<h4>${featureEvent.data.premise_address}</h4><div class="separator"></div>
				<h5 class = "lighter">License Type: ${featureEvent.data.license_type_code}</h5>
				<h5 class = "lighter">Serial number: ${featureEvent.data.serial_number}</h5>
				<h5 class = "lighter">Effective Date: ${featureEvent.data.license_issued_date}</h5>
        <h5 class = "lighter">Expiration Date: ${featureEvent.data.license_expiration_date}</h5>
        <h5 class = "lighter">Method of Operation: ${featureEvent.data.method_of_operation}</h5>
        <h5 class = "lighter">Days/Hours of Operation: ${featureEvent.data.days_hours_of_operation}</h5>
				<h5 class = "lighter"><a href= 'https://www.tran.sla.ny.gov/servlet/ApplicationServlet?pageName=com.ibm.nysla.data.publicquery.PublicQuerySuccessfulResultsPage&validated=true&serialNumber=${featureEvent.data.serial_number}&licenseType=${featureEvent.data.license_type_code}' target = '_blank'>Click here for more information about this license.</a></h5>`

    //adds CORS header to proxy request getting around errors
    const proxyurl = 'https://cors-anywhere.herokuapp.com/'

    //Query the city's Geoclient API to get the BIN of the building at the address listed in the SLA data. we will use this to collect the Certificate of Occupancy
    var url = `https://api.cityofnewyork.us/geoclient/v1/search.json?input=${featureEvent.data.premise_address} ${featureEvent.data.premise_zip}&app_id=${geoclient_id}&app_key=${geoclient_key}`

    var slaBIN = new Promise(function(resolve) {
      fetch(proxyurl + url)
        .then(function(response) {
          return response.json()
        })
        .then(function(address) {
          const results = address.results
          const response = results[0].response
          console.log(response)
          resolve(response.buildingIdentificationNumber)
        })
        .catch(function(error) {
          resolve(0)
        })
    })

    slaBIN.then(bin => {
      if (bin == 0) {
        content += `<div class="separator"></div><h4>Certificate of Occupancy</h4>><h5 class = "lighter">No BIN Found.</h5>`
      }
      //display a link to BISweb for the Certificate of Occupancy at that BIN
      else {
        const certURL = `http://a810-bisweb.nyc.gov/bisweb/COsByLocationServlet?requestid=1&allbin=${bin}`
        content += `<div class="separator"></div><h4>Certificate of Occupancy</h4><h5 class = "lighter">BIN: <a href= '${certURL}' target = '_blank'>${bin}</a></h5>`
      }

      //add source information
      source += `<div class="separator"></div><h6>Source: <a href='https://data.ny.gov/Economic-Development/Liquor-Authority-Quarterly-List-of-Active-Licenses/hrvs-fxs2'>Liquor Authority Quarterly List of Active Licenses.</a></h6><h6>Data is filtered to liquor licenses reviewed by the New York City Agency Office. Data is updated quarterly.</h6>`
      //set the info_box to display as block

      info.show({
        content,
        source,
        charts: []
      })
    })
  })

  client.getLeafletLayer().addTo(map)
</script>
