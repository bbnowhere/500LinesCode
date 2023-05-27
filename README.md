# 500LinesCode
## 500 Lines of code with explanation has been added.

<?php

/**
 * @file
 * Contains admissions.module.
 */

use Drupal\node\Entity\Node;
use Drupal\views\ViewExecutable;
use Drupal\Core\Access\AccessResult;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Component\Utility\Random;
use Drupal\Core\Routing\TrustedRedirectResponse;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Drupal\Core\Url;

/**
 * Implemented HOOK_preprocess()
 */


/* Explanation
The below function takes the year value from manually created global configuration file.The line $variables['gs_prefix'] = $gs_prefix; adds a new variable named 'gs_prefix' to the $variables array. This variable can be accessed in the template file where the preprocess function is being applied.The line \Drupal::service('renderer')->addCacheableDependency($variables, $gs_prefix); adds a cacheable dependency to the renderer service. The rendered output will be cached, but it will be invalidated and regenerated whenever the value of the $gs_prefix configuration setting is changed and saved.
*/

function admissions_preprocess(&$variables)
{

  $gs_prefix = \Drupal::config('admissions.global')->get('year');
  $variables['gs_prefix'] = $gs_prefix;
  // add the cache tag, so that the output gets invalidated when the config is saved
  \Drupal::service('renderer')->addCacheableDependency($variables, $gs_prefix);
}

/**
 * Implemented HOOk_entity_insert()
 */

function admissions_entity_insert(Drupal\Core\Entity\EntityInterface $entity)
{
  if (!empty($entity)) {

    // Database Connection
    $database = \Drupal::database();

    // while inserting application node set the entrance exam reference value
/*
 * Explanation 
 * It checks if the bundle of the inserted entity is 'applications' using the condition $entity->bundle() == 'applications'.
 * If the condition is true, it proceeds with the following steps:
 * It retrieves the value of the 'field_application_seq_no' field from the inserted entity using $entity->field_application_seq_no->getValue()[0]['value'] and assigns it to the variable $application_seq_no.
 * It creates a database query object using $query1 = $database->select('node__field_exam_application_seq_no', 'a'), selecting the 'entity_id' field from the 'node__field_exam_application_seq_no' table.
 * It adds conditions to the query to filter the rows with a matching 'field_exam_application_seq_no_value' value and a bundle value of 'entrance_exam_new'.
 * It executes the query using $query1->execute()->fetchField() and assigns the result to the variable $results_max.
 * It converts the value of $results_max to an integer using (int)($results_max) and assigns it to the variable $result.
 * It sets the value of the 'field_entrance_exam_refernce' field of the inserted entity to $result using $entity->set('field_entrance_exam_refernce', $result).
 * It saves the modified entity using $entity->save().
 * It loads the node object with the entity ID equal to $result using Node::load($result) and assigns it to the variable $exam_node_obj.
 * It checks if $exam_node_obj is not NULL.
 * If $exam_node_obj is not NULL, it sets the value of the 'field_application_reference' field of $exam_node_obj to the ID of the inserted entityusing $exam_node_obj->set('field_application_reference', $entity->id()).
 * It saves the modified $exam_node_obj using $exam_node_obj->save().
 */
    if ($entity->bundle() == 'applications') {

      $application_seq_no = $entity->field_application_seq_no->getValue()[0]['value'];
      $query1 = $database->select('node__field_exam_application_seq_no', 'a');
      $query1->fields('a', ['entity_id']);
      $query1->condition('a.bundle', 'entrance_exam_new');
      $query1->condition('a.field_exam_application_seq_no_value', $application_seq_no);
      $results_max = $query1->execute()->fetchField();
      $result = (int)($results_max);
      $entity->set('field_entrance_exam_refernce', $result);
      $entity->save();
      $exam_node_obj = Node::load($result);
      if ($exam_node_obj != NULL) {
        $exam_node_obj->set('field_application_reference', $entity->id());
        $exam_node_obj->save();
      }
    }
    // while inserting entrance exan node set the application reference value
    if ($entity->bundle() == 'entrance_exam_new') {
      $exam_application_seq_no = $entity->field_exam_application_seq_no->getValue()[0]['value'];
      $query2 = $database->select('node__field_application_seq_no', 'a');
      $query2->fields('a', ['entity_id']);
      $query2->condition('a.bundle', 'applications');
      $query2->condition('a.field_application_seq_no_value', $exam_application_seq_no);
      $results = $query2->execute()->fetchField();
      $result_application = (int)($results);
      $application_node_obj = Node::load($result_application);
      if ($application_node_obj != NULL) {
        $application_node_obj->set('field_entrance_exam_refernce', $entity->id());
        $application_node_obj->save();
        $entity->set('field_application_reference', $result_application);
        $entity->save();
      }
    }

/* The below code is written to send an email to the referees email id which are filled while filling the node of content type 
 * 'candidate information'. The below code stores the name and email id of the referee and pass the variables to the mail service
 * if the configuration is set to testing then it send the test mail else it takes the values in the variables and sends the mail to the 
 * referees. When referee recieves an email they get an email with unique session id in the URL to fill the form. When the system sends an 
 * email, the contents of an email with to and reciever information is recorded and saved in the node of log content type.*
 * This approach is used to send an email to the referees with the unique link which is accessible until the referee submission date (from and to is set in the system and automatically access gets denied once the last date of submissions gets over. By using this code referees can submit the form without login to the system.
 */

    if ($entity->bundle() == 'candidate_information') {
      $application_id = (int)(\Drupal::request()->query->get('application_id'));
      if ($application_id != NULL) {
        $node_obj = \Drupal::entityTypeManager()->getStorage('node')->load($application_id);
        $application_ref_id = $node_obj->field_application_reference->getValue()[0]['target_id'];
        $application_obj = \Drupal::entityTypeManager()->getStorage('node')->load($application_ref_id);
        $candidate_mail_id = $application_obj->field_application_email_id->getValue()[0]['value'];
        $mail_key = 'candidate_info';
        $node_obj->set('field_candidate_info_ref', $entity->id());
        $node_obj->save();
        $ref1_name = $entity->field_application_ref_1_name->getValue()[0]['value'];
        $ref1_mail = $entity->field_application_ref_1_email->getValue()[0]['value'];
        $ref2_name = $entity->field_application_ref_2_name->getValue()[0]['value'];
        $ref2_mail = $entity->field_application_ref_2_email->getValue()[0]['value'];
        $module = 'admissions';
        $key = 'referee';
        $webadmin_value = \Drupal::config('admissions.webadmin')->get('admissions_access_mail');
        if ($webadmin_value == 'Yes') {
          $params['subject'] = '[Testing Website]';
          $params['body'] = 'This is tesing mail.Please ignore.';
          $to = $candidate_mail_id . ',' . $ref1_mail . ',' . $ref2_mail;
          $mailManager = \Drupal::service('plugin.manager.mail');
          $langcode = \Drupal::currentUser()->getPreferredLangcode();
          $send = TRUE;
          $result = $mailManager->mail($module, $key, $to, $langcode, $params, NULL, $send);
        } else {
          $name = $node_obj->field_e_e_name->getValue()[0]['value'];
          $reference_no = $node_obj->field_exam_application_seq_no->getValue()[0]['value'];
          $ref1_session_val = generateRandomString();
          $ref2_session_val = generateRandomString();
          $entity->set('field_ref1_session', $ref1_session_val);
          $entity->set('field_ref2_session', $ref2_session_val);
          $entity->set('field_entrance_exam_ref', $application_id);
          $entity->set('title', $name . ' has Submitted');
          $entity->save();
          $template_name = 'Candidate Referee';
          $host = \Drupal::request()->getSchemeAndHttpHost();
          //$base_url = \Drupal::request()->getBaseUrl();
	  $base_url = 'www.ncbs.res.in/admissions/GS21';
	  $host = 'https://';
          $body_ref1 = $host . '' . $base_url . '/node/add/referee?session1=' . $ref1_session_val . '&candidate_id=' . $entity->id();
          $body_ref2 = $host . '' . $base_url . '/node/add/referee?session2=' . $ref2_session_val . '&candidate_id=' . $entity->id();
          $candidate_template_name = 'Email sent to candidate after submission';
          $candidate_submission_mail = \Drupal::service('admissions.load_email_template')->getEmailContent($mail_key, $candidate_mail_id, $body = '', $no = '', $name, $module, $candidate_template_name, $name1 = '', $academic_office_mail = '', $academic_head_mail = '', $address = '', $mobile = '');
          $template_data1 = \Drupal::service('admissions.load_email_template')->getEmailContent($key, $ref1_mail, $body_ref1, $reference_no, $name, $module, $template_name, $ref1_name, $academic_office_mail = '', $academic_head_mail = '', $address = '', $mobile = '');
          $template_data2 = \Drupal::service('admissions.load_email_template')->getEmailContent($key, $ref2_mail, $body_ref2, $reference_no, $name, $module, $template_name, $ref2_name, $academic_office_mail = '', $academic_head_mail = '', $address = '', $mobile = '');
          $candidate_obj_values = \Drupal::entityQuery('node')->condition('type', 'log')->condition('title', $ref1_name . ' Refered by ' . $name)->execute();
          $candidate_node_not_exists = empty($candidate_obj_values);
          if ($candidate_node_not_exists) {
            $log_node = \Drupal::entityTypeManager()->getStorage('node')->create([
              'type' => 'log',
              'title' => $name . ' has Uploaded Referee Details',
              'field_all_entities_type' => 'Candidate Submission Mail',
              'field_user' => 0,
              'field_all_entities_message' => $name . ' has Uploaded Referee Details',
              'field_email_subject' => $candidate_submission_mail['subject'],
              'field_email_body' => ['value' => $candidate_submission_mail['body'], 'format' => 'full_html'],
              'field_email_sent_to' => $candidate_mail_id,
              'field_candidate_seq_no' => $reference_no
            ]);
            $log_node->save();
          } else {
            /*if node exist update the node*/
            $log_nid = reset($candidate_obj_values);
            $log_node = \Drupal::entityTypeManager()->getStorage('node')->load($log_nid);
            $log_node->set('title', $name . ' has Uploaded Referee Details');
            $log_node->set('field_all_entities_type', 'Candidate Submission Mail');
            $log_node->set('field_user', 0);
            $log_node->set('field_all_entities_message', $name . ' has Uploaded Referee Details');
            $log_node->set('field_email_subject', $candidate_submission_mail['subject']);
            $log_node->set('field_email_body', $candidate_submission_mail['body']);
            $log_node->set('field_email_sent_to', $candidate_mail_id);
            $log_node->set('field_candidate_seq_no', $reference_no);
            $log_node->save();
          }
          $node_values = \Drupal::entityQuery('node')->condition('type', 'log')->condition('title', $ref1_name . ' Refered by ' . $name)->execute();
          $node_not_exists = empty($node_values);
          if ($node_not_exists) {
            $node = \Drupal::entityTypeManager()->getStorage('node')->create([
              'type' => 'log',
              'title' => $ref1_name . ' Refered by ' . $name,
              'field_all_entities_type' => 'Candidate Referee',
              'field_user' => 0,
              'field_all_entities_message' => $ref1_name . ' Refered by ' . $name,
              'field_email_subject' => $template_data1['subject'],
              'field_email_body' => ['value' => $template_data1['body'], 'format' => 'full_html'],
              'field_email_sent_to' => $ref1_mail,
              'field_candidate_seq_no' => $reference_no
            ]);
            $node->save();
          } else {
            /*if node exist update the node*/
            $nid = reset($node_values);
            $node = \Drupal::entityTypeManager()->getStorage('node')->load($nid);
            $node->set('title', $ref1_name . ' Refered by ' . $name);
            $node->set('field_all_entities_type', 'Candidate Referee');
            $node->set('field_user', 0);
            $node->set('field_all_entities_message', $ref1_name . ' Refered by ' . $name);
            $node->set('field_email_subject', $template_data1['subject']);
            $node->set('field_email_body', $template_data1['body']);
            $node->set('field_email_sent_to', $ref1_mail);
            $node->set('field_candidate_seq_no', $reference_no);
            $node->save();
          }
          $node1_values = \Drupal::entityQuery('node')->condition('type', 'log')->condition('title', $ref2_name . ' Refered by ' . $name)->execute();
          $node1_not_exists = empty($node1_values);
          if ($node1_not_exists) {
            $node1 = \Drupal::entityTypeManager()->getStorage('node')->create([
              'type' => 'log',
              'title' => $ref2_name . ' Refered by ' . $name,
              'field_all_entities_type' => 'Candidate Referee',
              'field_user' => 0,
              'field_all_entities_message' => $ref2_name . ' Refered by ' . $name,
              'field_email_subject' => $template_data2['subject'],
              'field_email_body' => ['value' => $template_data2['body'], 'format' => 'full_html'],
              'field_email_sent_to' => $ref2_mail,
              'field_candidate_seq_no' => $reference_no
            ]);
            $node1->save();
          } else {
            /*if node exist update the node*/
            $nid1 = reset($node1_values);
            $node1 = \Drupal::entityTypeManager()->getStorage('node')->load($nid1);
            $node1->set('title', $ref2_name . ' Refered by ' . $name);
            $node1->set('field_all_entities_type', 'Candidate Referee');
            $node1->set('field_user', 0);
            $node1->set('field_all_entities_message', $ref2_name . ' Refered by ' . $name);
            $node1->set('field_email_subject', $template_data2['subject']);
            $node1->set('field_email_body', $template_data2['body']);
            $node1->set('field_email_sent_to', $ref2_mail);
            $node1->set('field_candidate_seq_no', $reference_no);
            $node1->save();
/*
 * The below code sends an email to the candidate when their referees successfully submits referee form recieve to them via email. 
 * The code further records the logs in the nodes of logs content type which is recorded so that auditors of the site could check the logs
 * whether the candidate recieves an email confirmation or not.	    
 */	
if ($entity->bundle() == 'referee') {
      $current_url_referee = \Drupal::request()->getRequestUri();
      if (str_contains($current_url_referee, 'session1')) {
        $candidate_id = (int)(\Drupal::request()->query->get('candidate_id'));
        if ($candidate_id != NULL) {
          $candidate_obj = \Drupal::entityTypeManager()->getStorage('node')->load($candidate_id);
          if ($candidate_obj != NULL) {
            $ent_exam_id = $candidate_obj->field_entrance_exam_ref->getValue()[0]['target_id'];
            $exam_obj = \Drupal::entityTypeManager()->getStorage('node')->load($ent_exam_id);
            $app_id = $exam_obj->field_application_reference->getValue()[0]['target_id'];
            $app_obj = \Drupal::entityTypeManager()->getStorage('node')->load($app_id);
            $referee1_name = $candidate_obj->field_application_ref_1_name->getValue()[0]['value'];
            $entrance_exam_value =  $candidate_obj->field_entrance_exam_ref->getValue()[0]['target_id'];
            $entrance_exam_obj = \Drupal::entityTypeManager()->getStorage('node')->load($entrance_exam_value);
            $candidate_name = $entrance_exam_obj->field_e_e_name->getValue()[0]['value'];
            $candidate_seq_no = $entrance_exam_obj->field_exam_application_seq_no->getValue()[0]['value'];
            $candidate_obj->set('field_ref1_node_ref', $entity->id());
            $candidate_obj->save();
            $entity->set('field_candidate_info', $candidate_id);
            $entity->set('title', $referee1_name . ' Refered by ' . $candidate_name);
            $entity->save();
            $candidate_email = $app_obj->field_application_email_id->getValue()[0]['value'];
            $candidate_name = $app_obj->field_application_full_name->getValue()[0]['value'];
            $sequence_no = $app_obj->field_application_seq_no->getValue()[0]['value'];
            $referee1_name = $candidate_obj->field_application_ref_1_name->getValue()[0]['value'];
            $referee1_email = $candidate_obj->field_application_ref_1_email->getValue()[0]['value'];
            $body = '';
            $module = 'admissions';
            $mail_key = 'referee';
            $template_name = 'Final submission of the Referee';
            $log_node_title1 = $referee1_name . ' has Uploaded Final Referee Report';
            $referee_submission_mail = \Drupal::service('admissions.load_email_template')->getEmailContent($mail_key, $candidate_email, $body = '', $sequence_no, $candidate_name, $module, $template_name, $referee1_name, $academic_office_mail = '', $academic_head_mail = '', $address = '', $candidate_id);
            $ref1_node_obj = \Drupal::entityQuery('node')->condition('type', 'log')->condition('title', $referee1_name . ' has Uploaded Final Referee Report')->execute();
            $ref1_node_not_exists = empty($ref1_node_obj);
            if ($ref1_node_not_exists) {
              $ref1_node = \Drupal::entityTypeManager()->getStorage('node')->create([
                'type' => 'log',
                //'title' => $referee1_name . ' Refered by ' . $candidate_name . ' has Submitted',
                'title' => $referee1_name . ' has Uploaded Final Referee Report',
                'field_all_entities_type' => 'Referee Submitted',
                'field_all_entities_type' => 'Referee Submitted',
                'field_user' => 0,
                'field_email_subject' => $referee_submission_mail['params']['subject'],
                'field_email_body' => ['value' => $referee_submission_mail['params']['body'], 'format' => 'full_html'],
                'field_email_sent_to' => $candidate_email,
                'field_all_entities_message' => $referee1_name . ' Refered by ' . $candidate_name . ' has Submitted Final Report',
                'field_candidate_seq_no' => $candidate_seq_no
              ]);
              $ref1_node->save();
	      \Drupal::messenger()->addMessage($referee1_name . ' Refered by ' . $candidate_name . ' has submitted.');
            } else {
              /*if node exist update the node*/
              $ref1_nid = reset($ref1_node_obj);
              $ref1_node = \Drupal::entityTypeManager()->getStorage('node')->load($ref1_nid);
              $ref1_node->set('title', $referee1_name . ' Refered by ' . $candidate_name . ' has Submitted');
              $ref1_node->set('field_all_entities_type', 'Referee Submitted');
              $ref1_node->set('field_user', 0);
              $ref1_node->set('field_all_entities_message', $referee1_name . ' Refered by ' . $candidate_name . ' has Submitted Final Report');
              $ref1_node->set('field_email_subject', $referee_submission_mail['params']['subject']);
              $ref1_node->set('field_email_body', $referee_submission_mail['params']['body']);
              $ref1_node->set('field_email_sent_to', $candidate_email);
              $ref1_node->set('field_candidate_seq_no', $candidate_seq_no);
              $ref1_node->save();
	      \Drupal::messenger()->addMessage($referee1_name . ' Refered by ' . $candidate_name . ' has submitted.');
            }
          }
        }
      }
      if (str_contains($current_url_referee, 'session2')) {
        $candidate_id = (int)(\Drupal::request()->query->get('candidate_id'));
        if ($candidate_id != NULL) {
          $candidate_obj1 = \Drupal::entityTypeManager()->getStorage('node')->load($candidate_id);
          if ($candidate_obj1 != NULL) {
            $referee2_name = $candidate_obj1->field_application_ref_2_name->getValue()[0]['value'];
            $entrance_exam_value1 =  $candidate_obj1->field_entrance_exam_ref->getValue()[0]['target_id'];
            $entrance_exam_obj1 = \Drupal::entityTypeManager()->getStorage('node')->load($entrance_exam_value1);
            $candidate_name1 = $entrance_exam_obj1->field_e_e_name->getValue()[0]['value'];
            $candidate_seq_no1 = $entrance_exam_obj1->field_exam_application_seq_no->getValue()[0]['value'];
            $candidate_obj1->set('field_ref2_node_ref', $entity->id());
            $candidate_obj1->save();
            $entity->set('field_candidate_info', $candidate_id);
            $entity->set('title', $referee2_name . ' Refered by ' . $candidate_name1);
            $entity->save();
            $app_id1 = $entrance_exam_obj1->field_application_reference->getValue()[0]['target_id'];
            $app_obj1 = \Drupal::entityTypeManager()->getStorage('node')->load($app_id1);
                        $candidate_email = $app_obj1->field_application_email_id->getValue()[0]['value'];
                        $candidate_name = $app_obj1->field_application_full_name->getValue()[0]['value'];
                        $sequence_no = $app_obj1->field_application_seq_no->getValue()[0]['value'];
                        $referee2_name = $candidate_obj1->field_application_ref_2_name->getValue()[0]['value'];
                        $referee2_email = $candidate_obj1->field_application_ref_2_email->getValue()[0]['value'];
                        $body = '';
                        $module = 'admissions';
                        $mail_key = 'referee';
                        $template_name = 'Final submission of the Referee';
                        $log_node_title2 = $referee2_name . ' has Uploaded Final Referee Report';
                        $referee_submission_mail1 = \Drupal::service('admissions.load_email_template')->getEmailContent($mail_key, $candidate_email, $body = '', $sequence_no, $candidate_name, $module, $template_name, $referee2_name, $academic_office_mail = '', $academic_head_mail = '', $address = '', $candidate_id);
                        $ref2_node_obj = \Drupal::entityQuery('node')->condition('type', 'log')->condition('title', $referee2_name . ' has Uploaded Final Referee Report')->execute();
                        $ref2_node_not_exists = empty($ref2_node_obj);
                        if ($ref2_node_not_exists) {
                          $ref1_node = \Drupal::entityTypeManager()->getStorage('node')->create([
                            'type' => 'log',
                            'title' => $referee2_name . ' has Uploaded Final Referee Report',
                            'field_all_entities_type' => 'Referee Submitted',
                            'field_user' => 0,
                            'field_email_subject' => $referee_submission_mail1['params']['subject'],
                            'field_email_body' => ['value' => $referee_submission_mail1['params']['body'], 'format' => 'full_html'],
                            'field_email_sent_to' => $candidate_email,
                            'field_all_entities_message' => $referee2_name . ' Refered by ' . $candidate_name . ' has Submitted Final Report',
                            'field_candidate_seq_no' => $candidate_seq_no1
                          ]);
                          $ref1_node->save();
		          \Drupal::messenger()->addMessage($referee2_name . ' Refered by ' . $candidate_name1 . ' has submitted.');
                        } else {
                          /*if node exist update the node*/
                          $ref2_nid = reset($ref2_node_obj);
                          $ref2_node = \Drupal::entityTypeManager()->getStorage('node')->load($ref2_nid);
                          $ref2_node->set('title', $referee2_name . ' Refered by ' . $candidate_name1 . ' has Submitted');
                          $ref2_node->set('field_all_entities_type', 'Referee Submitted');
                          $ref2_node->set('field_user', 0);
                          //$ref2_node->set('field_all_entities_message', $referee2_name . ' Refered by ' . $candidate_name1 . ' has Submitted');
                          $ref2_node->set('field_all_entities_message', $referee1_name . ' Refered by ' . $candidate_name . ' has Submitted Final Report');
                          $ref2_node->set('field_email_subject', $referee_submission_mail1['params']['subject']);
                          $ref2_node->set('field_email_body', $referee_submission_mail1['params']['body']);
                          $ref2_node->set('field_email_sent_to', $candidate_email);
                          $ref2_node->set('field_candidate_seq_no', $candidate_seq_no1);
                          $ref2_node->save();
			  \Drupal::messenger()->addMessage($referee2_name . ' Refered by ' . $candidate_name1 . ' has submitted.');
            }
          }
        }
      }
    }

/* The below code automatically filters the view by comparing the value from the global configuration. If the value is more than which is 
 * configured in the global then it displays the result.
 * This code is written to display the list of qualified candidates candidates after the cut-off. This approach is used so that site admin can set the cut-off marks in global conf. and see the list of qualified candidates by viewing the specific view page.
 * */
function admissions_views_pre_view(ViewExecutable $view, $display_id, array &$args)
{
  if (($view->id() == 'ph_d') || ($view->id() == 'phd_screening')) {
    $view->display_handler->options['filters']['field_total_score_value']['value']['value'] = \Drupal::config('admissions.phd')->get('admissions_phd_cutoff');
    $view->display_handler->options['filters']['field_gate_score_value']['value']['value'] = \Drupal::config('admissions.phd')->get('phd_gate_cutoff');
  }
  if ($view->id() == 'qualified_candidates_in_int_ph_d' || $view->id() == 'int_phd_screening') {
    $view->display_handler->options['filters']['field_total_score_value']['value']['value'] = \Drupal::config('admissions.intphd')->get('admissions_intphd_cutoff');
    $view->display_handler->options['filters']['field_gate_score_value']['value']['value'] = \Drupal::config('admissions.intphd')->get('intphd_gate_cutoff');
  }
  if ($view->id() == 'wildlife_list') {
    $view->display_handler->options['filters']['field_total_score_value']['value']['value'] = \Drupal::config('admissions.wildlife')->get('admissions_wild_cutoff');
    $view->display_handler->options['filters']['field_gate_score_value']['value']['value'] = \Drupal::config('admissions.wildlife')->get('wildlife_gate_cutoff');
  }
  if ($view->id() == 'qualified_candidates_in_both_int_phd_and_m_sc' || $view->id() == 'both_int_phd_and_m_sc_screening') {
    $filters = $view->display_handler->getOption('filters');  
    $filters['field_total_score_value']['value']['value'] = \Drupal::config('admissions.bothintphd')->get('admissions_bothintphd_cutoff');
    $filters['field_gate_score_value']['value']['value'] = \Drupal::config('admissions.bothintphd')->get('bothintphd_gate_cutoff');
    $view->display_handler->overrideOption('filters', $filters);
  }
  if ($view->id() == 'qualified_candidates_in_m_sc_biology' || $view->id() == 'm_sc_screening_') {
    $view->display_handler->options['filters']['field_total_score_value']['value']['value'] = \Drupal::config('admissions.msc')->get('admissions_msc_cutoff');
    $view->display_handler->options['filters']['field_gate_score_value']['value']['value'] = \Drupal::config('admissions.msc')->get('msc_gate_cutoff');
  }
}

/*
 * Implements hook_mail().
 */
/* The below function implements the hook mail. 
 * The function takes the three important parameters which are key ,message and params. Based on the key it takes the params and subject.
 * The function is called in the hook entity insert based on the condition.
 * */

function admissions_mail($key, &$message, $params)
{
  if (($key == 'intimate_students') || ($key == 'candidate_info') ||  ($key == 'referee') ||  ($key == 'intimate_commitees') ||  ($key == 'intimate_interview1') ||  ($key == 'intimate_interview2') || ($key == 'intimate_students_interview_phd') || ($key == 'intimate_students_interview_intphd') || ($key == 'intimate_selected_phd') || ($key == 'intimate_selected_int_phd') || ($key == 'intimate_selected_both_phd')) {
    $message['from'] = \Drupal::config('system.site')->get('mail');
    $message['subject'] = $params['subject'];
    $message['body'][] = $params['body'];
  }
}

/* The below function is written to create an access based on the roles or session id and on the last date of submission.
 * The access is allowed to the user if the user has role admin or phd_admin.
 * If the link has correct session id,node id and submission date is not over then the link is accessible to the users.
 * */

function admissions_entity_create_access(\Drupal\Core\Session\AccountInterface $account, array $context, $entity_bundle)
{
  $current_user_role = \Drupal::currentUser()->getRoles();
  if (in_array('admin', $current_user_role) || in_array('phd_admin', $current_user_role)) {
  	return AccessResult::allowed();
  }
  // giving access permission for creating content to candidate for uploading referee details
  if ($entity_bundle == 'candidate_information') {
            return AccessResult::neutral();
    $current_datetime = new DateTime();
    $application_id = (int)(\Drupal::request()->query->get('application_id'));
    $session_url_val = \Drupal::request()->query->get('session');
    if ($application_id != NULL) {
      $node_obj = \Drupal::entityTypeManager()->getStorage('node')->load($application_id);
      if ($node_obj != NULL) {
        \Drupal::service("page_cache_kill_switch")->trigger();
        if (($node_obj->field_session_key_cand_ref->getValue()[0]['value'] == $session_url_val) && ($application_id == $node_obj->id()) && (isset($node_obj->field_candidate_info_ref->getValue()[0]['target_id']) == NULL) && ((strtotime($current_datetime->format('Y-m-d H:i:s'))) < (strtotime(\Drupal::config('admissions.global')->get('candidate_submission_date'))))) {
          return AccessResult::allowed();
        } else {
          return AccessResult::neutral();
        }
      }
    }
  }
// giving access permission for creating content to referees for entering details
  if ($entity_bundle == 'referee') {
//Comment this to give access to referee
             return AccessResult::neutral();
    $current_datetime = new DateTime();
    $current_url = \Drupal::request()->getRequestUri();
    \Drupal::service("page_cache_kill_switch")->trigger();
    if (str_contains($current_url, 'session1')) {
      $ref1_session = \Drupal::request()->query->get('session1');
      $candidate_id = (int)(\Drupal::request()->query->get('candidate_id'));
      if ($candidate_id != NULL) {
        $node_obj = \Drupal::entityTypeManager()->getStorage('node')->load($candidate_id);
        if ($node_obj != NULL) {
          if (($node_obj->field_ref1_session->getValue()[0]['value'] == $ref1_session) && ($candidate_id == $node_obj->id()) && (isset($node_obj->field_ref1_node_ref->getValue()[0]['target_id']) == NULL) && ((strtotime($current_datetime->format('Y-m-d H:i:s'))) < (strtotime(\Drupal::config('admissions.global')->get('referee_submission_date'))))) {
            return AccessResult::allowed();
          } else {
            return AccessResult::neutral();
          }
        }
      }
    }
    if (str_contains($current_url, 'session2')) {
      $ref2_session = \Drupal::request()->query->get('session2');
      $candidate_id = (int)(\Drupal::request()->query->get('candidate_id'));
      if ($candidate_id != NULL) {
        $node_obj = \Drupal::entityTypeManager()->getStorage('node')->load($candidate_id);
        if ($node_obj != NULL) {
          if (($node_obj->field_ref2_session->getValue()[0]['value'] == $ref2_session) && ($candidate_id == $node_obj->id()) && (isset($node_obj->field_ref2_node_ref->getValue()[0]['target_id']) == NULL)) {
            return AccessResult::allowed();
          } else {
            return AccessResult::neutral();
          }
        }
      }
    }
  }
  if ($entity_bundle == 'shortlist') {
    $candidate_id = (int)(\Drupal::request()->query->get('id'));
    if ($candidate_id != NULL) {
      $candidate_node_obj = \Drupal::entityTypeManager()->getStorage('node')->load($candidate_id);
      if ($candidate_node_obj != NULL) {
        if (\Drupal::config('admissions.global')->get('admissions_screening_types') == 'Screening') {
          $current_user_role = \Drupal::currentUser()->getRoles();
          foreach ($current_user_role as $current_user) {
            if (strpos($current_user, 'admin') !== FALSE) {
              return AccessResult::allowed();
            }
          }
          $shortlist_obj = $candidate_node_obj->field_shortlist_reference->getValue();
          if ($shortlist_obj != NULL) {
            foreach ($shortlist_obj as $shortlist_ids) {
              $shortlist_node_ids = $shortlist_ids['target_id'];
              if ($shortlist_node_ids != NULL) {
                $shortlist_obj_ids = \Drupal::entityTypeManager()->getStorage('node')->load($shortlist_node_ids);
                if ($shortlist_obj_ids != NULL) {
                  $user_obj[] = $shortlist_obj_ids->uid->getValue()[0]['target_id'];
                }
              }
            }
            if (in_array(\Drupal::currentUser()->id(), $user_obj)) {
              return AccessResult::neutral();
            }
          } else {
            if ($candidate_node_obj->field_screening_commitee->getValue() != NULL) {
              $screening_commitee_node = (int)($candidate_node_obj->field_screening_commitee->getValue()[0]['target_id']);
              if ($screening_commitee_node != NULL) {
                $screening_commitee_obj = \Drupal::entityTypeManager()->getStorage('node')->load($screening_commitee_node);
                if ($screening_commitee_obj != NULL) {
                  if ($screening_commitee_obj->field_member != NULL) {
                    $screening_members_obj = $screening_commitee_obj->field_member->getValue();
                    foreach ($screening_members_obj as $screening_members) {
                      $screening_users[] = (int)($screening_members['target_id']);
                    }
                    if (in_array(\Drupal::currentUser()->id(), $screening_users)) {
                      return AccessResult::allowed();
                    } else {
                      return AccessResult::neutral();
                    }
                  }
                }
              }
            }
          }
        }
        if (\Drupal::config('admissions.global')->get('admissions_screening_types') == 'Interview1') {
          $current_user_role = \Drupal::currentUser()->getRoles();
          foreach ($current_user_role as $current_user) {
            if (strpos($current_user, 'admin') !== FALSE) {
              return AccessResult::allowed();
            }
          }
          $interview1_shortlist_obj = $candidate_node_obj->field_interview1_shortlist_refer->getValue();
          if ($interview1_shortlist_obj != NULL) {
            foreach ($interview1_shortlist_obj as $interview1_shortlist_ids) {
              $interview1_shortlist_node_ids = $interview1_shortlist_ids['target_id'];
              if ($interview1_shortlist_node_ids != NULL) {
                $interview1_shortlist_obj_ids = \Drupal::entityTypeManager()->getStorage('node')->load($interview1_shortlist_node_ids);
                if ($interview1_shortlist_obj_ids != NULL) {
                  $interview1_user_ids[] = $interview1_shortlist_obj_ids->uid->getValue()[0]['target_id'];
                }
              }
            }
            if (in_array(\Drupal::currentUser()->id(), $interview1_user_ids)) {
              return AccessResult::neutral();
            }
          } else {
            if ($candidate_node_obj->field_interview1_commitee->getValue() != NULL) {
              $interview1_commitee_node = (int)($candidate_node_obj->field_interview1_commitee->getValue()[0]['target_id']);
              if ($interview1_commitee_node != NULL) {
                $interview1_commitee_obj = \Drupal::entityTypeManager()->getStorage('node')->load($interview1_commitee_node);
                if ($interview1_commitee_obj != NULL) {
                  if ($interview1_commitee_obj->field_member != NULL) {
                    $interview1_members_obj = $interview1_commitee_obj->field_member->getValue();
                    foreach ($interview1_members_obj as $interview1_members) {
                      $interview1_users_ids[] = (int)($interview1_members['target_id']);
                    }
                    if (in_array(\Drupal::currentUser()->id(), $interview1_users_ids)) {
                      return AccessResult::allowed();
                    } else {
                      return AccessResult::neutral();
                    }
                  }
                }
              }
            }
          }
        }
        if (\Drupal::config('admissions.global')->get('admissions_screening_types') == 'Interview2') {
          $current_user_role = \Drupal::currentUser()->getRoles();
          foreach ($current_user_role as $current_user) {
            if (strpos($current_user, 'admin') !== FALSE) {
              return AccessResult::allowed();
            }
          }
          $interview2_shortlist_obj = $candidate_node_obj->field_interview2_shortlist_refer->getValue();
          if ($interview2_shortlist_obj != NULL) {
            foreach ($interview2_shortlist_obj as $interview2_shortlist_ids) {
              $interview2_shortlist_node_ids = $interview2_shortlist_ids['target_id'];
              if ($interview2_shortlist_node_ids != NULL) {
                $interview2_shortlist_obj_ids = \Drupal::entityTypeManager()->getStorage('node')->load($interview2_shortlist_node_ids);
                if ($interview2_shortlist_obj_ids != NULL) {
                  $interview2_user_ids[] = $interview2_shortlist_obj_ids->uid->getValue()[0]['target_id'];
                }
              }
            }
            if (in_array(\Drupal::currentUser()->id(), $interview2_user_ids)) {
              return AccessResult::neutral();
            }
          } else {
            if ($candidate_node_obj->field_interview2_commitee->getValue() != NULL) {
              $interview2_commitee_node = (int)($candidate_node_obj->field_interview2_commitee->getValue()[0]['target_id']);
              if ($interview2_commitee_node != NULL) {
                $interview2_commitee_obj = \Drupal::entityTypeManager()->getStorage('node')->load($interview2_commitee_node);
                if ($interview2_commitee_obj != NULL) {
                  if ($interview2_commitee_obj->field_member != NULL) {
                    $interview2_members_obj = $interview2_commitee_obj->field_member->getValue();
                    foreach ($interview2_members_obj as $interview2_members) {
                      $interview2_users_ids[] = (int)($interview2_members['target_id']);
                    }
                    if (in_array(\Drupal::currentUser()->id(), $interview2_users_ids)) {
                      return AccessResult::allowed();
                    } else {
                      return AccessResult::neutral();
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}


/* The below code allows viewing access to the anonymous users for node of content type 'instructions' */

function admissions_node_access(\Drupal\node\NodeInterface $node, $op, \Drupal\Core\Session\AccountInterface $account)
{
  if ($op == 'view' && $account->getRoles()[0] == 'anonymous' && $node->bundle() != 'instructions') {
    return AccessResult::forbidden();
  }
}

/**
 * Implements hook_form_alter().
 * @param $form
 * @param FormStateInterface $formState
 * @param $form_id
 */


/* The below function is used for altering the elements of node form. 
 *
 * */
function admissions_form_alter(&$form, FormStateInterface $formState, $form_id)
{
  global $base_url;
  if ($form_id == 'user_login_form') {
    $form['name']['#weight'] = 1;
    $form['pass']['#weight'] = 2;
    $form['forgot-password'] = [
      '#markup' => 'link',
      '#weight' => 99,
      '#markup' => '<div class="forgot-password mb-4 text-align-right"> <a href="https://ncbs.res.in/admissions/GS21/user/password">Forgot Password?</a></div>',
    ];
  }
  if ($form_id == 'node_log_edit_form') {
    $form['field_email_body']['#disabled'] = 'disabled';
    $form['field_email_sent_to']['#disabled'] = 'disabled';
    $form['field_email_subject']['#disabled'] = 'disabled';
    $form['field_all_entities_message']['#disabled'] = 'disabled';
    $form['field_all_entities_type']['#disabled'] = 'disabled';
    $form['field_user']['#disabled'] = 'disabled';
  }
  $current_user_role = \Drupal::currentUser()->getRoles();
  if ((!in_array('admin', $current_user_role)) && ($form_id != 'csvimport_form')) {
    if (strpos($form['#id'], 'views') !== false) {
    } else {
      $request = \Drupal::request();
      $route = $request->attributes->get(\Symfony\Cmf\Component\Routing\RouteObjectInterface::ROUTE_OBJECT);
      $title = \Drupal::service('title_resolver')->getTitle($request, $route);
      $revision = FALSE;
      $submit_value = $form['actions']['submit']['#value'];
      if ($form_id == 'node_candidate_information_form') {
        $title = 'Candidate Information Form';
        $submit_value = 'Submit';
        $form['#entity_builders'][] = 'setting_node_title';
      }
      if ($form_id == 'node_referee_form') {
        $title = 'Referee Form';
        $submit_value = 'Submit';
        $candidate_name = NULL;
        $candidate_id = (int)(\Drupal::request()->query->get('candidate_id'));
        if ($candidate_id != NULL) {
          $candidate_node_obj = \Drupal::entityTypeManager()->getStorage('node')->load($candidate_id);
          if ($candidate_node_obj != NULL) {
            $entrance_exam_obj =  $candidate_node_obj->field_entrance_exam_ref->getValue()[0]['target_id'];
            $candidate_name = (\Drupal::entityTypeManager()->getStorage('node')->load($entrance_exam_obj))->field_e_e_name->getValue()[0]['value'];
          }
        }
        $form['field_candidate_how_long']['widget'][0]['value']['#title'] = 'Since when (which year, and how long) and in what capacity have you known the candidate ' . $candidate_name . ' ?';
        $form['#entity_builders'][] = 'setting_node_title';
      }
      if ($form_id == 'node_shortlist_form') {
        $candidate_id = (int)(\Drupal::request()->query->get('id'));
        $candidate_name = NULL;
        $candidate_seq_no = NULL;
        if ($candidate_id != NULL) {
          $candidate_node_obj = \Drupal::entityTypeManager()->getStorage('node')->load($candidate_id);
          if ($candidate_node_obj != NULL) {
            $candidate_name = $candidate_node_obj->field_e_e_name->getValue()[0]['value'];
            $candidate_seq_no = $candidate_node_obj->field_exam_application_seq_no->getValue()[0]['value'];
          }
        }
        $title = 'Recommendation for ' . $candidate_name . '(' . $candidate_seq_no . ')';
        $submit_value = 'Submit';
        $previous_url = \Drupal::request()->headers->get('referer');
        $form['product'] = [
          '#type' => 'hidden',
          '#value' => $previous_url,
        ];
        $form['#entity_builders'][] = 'setting_node_title';
        $form['actions']['submit']['#submit'][] = 'redirecting_form';
      }
      $form['#title'] = $title;
      $form['revision_information']['#access'] = $revision;
      $form['actions']['submit']['#value'] = $submit_value;
      // Retrieve and remove all messages.
      $messages = drupal_get_messages('error');
      // Retrieve all messages, without removing them.
      $messages = drupal_get_messages(NULL, FALSE); // or
      $messages = drupal_set_message();
    }
  }
}

/* This function is written to generate the random number, which is further sent generates and assign unique session which generated session id which is used in the above code for generating the URL for the candidates and the referees.
 *
 * */
function generateRandomString($length = 20)
{
  $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  $charactersLength = strlen($characters);
  $randomString = '';
  for ($i = 0; $i < $length; $i++) {
    $randomString .= $characters[rand(0, $charactersLength - 1)];
  }
  return $randomString;
}

/* The below function is used for redirected to the view URL */

function redirecting_form(array $form, FormStateInterface $form_state)
{
  $view_url = $form_state->getUserInput()['product'];
  return $form_state->setResponse(new TrustedRedirectResponse($view_url));
}

/* This function adds customed exposed form filter,which is generated from the getOptions function. 
 * This is function is written to create list option for filtering.
 */

function admissions_form_views_exposed_form_alter(&$form, \Drupal\Core\Form\FormStateInterface $form_state, $form_id)
{
  $view = $form_state->getStorage('view');
  if ($form_id == 'views_exposed_form') {
    if (isset($form['field_total_score_value'])) {
      $form['field_total_score_value']['#size'] = 2;
    } elseif (isset($form['field_total_score_value_1'])) {
      $form['field_total_score_value_1']['#size'] = 2;
    }
    if (isset($form['title_1'])) {
      $form['title_1']['#type'] = 'select';
      $form['title_1']['#size'] = null;
      $form['title_1']['#options'] = getOptions('Screening');
    }
    if (isset($form['title_2'])) {
      $form['title_2']['#type'] = 'select';
      $form['title_2']['#size'] = null;
      $form['title_2']['#options'] = getOptions('Interview1 Panel');
    }
    if (isset($form['title_3'])) {
      $form['title_3']['#type'] = 'select';
      $form['title_3']['#size'] = null;
      $form['title_3']['#options'] = getOptions('Interview2 Panel');
    }
  }
}

/* The usage of the function is mentioned above. */

function getOptions($category)
{
  $obj = \Drupal::entityTypeManager()->getStorage('node')->loadByProperties(['type' => 'commity', 'field_category' => $category]);
  $options = [];
  if ($obj != NULL) {
    foreach ($obj as $ids) {
      $options[''] = '- Any - ';
      $options[$ids->getTitle()] =  $ids->getTitle();
    }
  } else {
    $options = '';
  }
  return $options;
}
