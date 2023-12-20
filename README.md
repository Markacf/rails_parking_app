# rails_parking_app
Assessment exam
Explanation:
Model: ParkingSpace
# app/models/parking_space.rb
class ParkingSpace < ApplicationRecord
  has_many :entrances, dependent: :destroy
  has_many :parking_slots, dependent: :destroy, counter_cache: true

  def adjust_slots(number_of_slots)
    current_slots = parking_slots.count
    number_of_slots = number_of_slots[:number_of_slots]
    puts current_slots
    puts number_of_slots
    if number_of_slots >= current_slots
      (number_of_slots - current_slots).times do
        parking_slots.create(size: 0) # You can adjust the size as needed
      end
    else
      parking_slots.destroy_all(limit: current_slots - number_of_slots)
    end
  end
end

In the ParkingSpace model, the adjust_slots method is defined. This method takes the number_of_slots as a parameter and adjusts the number of parking slots for the parking space accordingly.

The method first retrieves the current number of slots by counting the associated parking_slots. It then compares the number_of_slots parameter with the current number of slots. If the number_of_slots is greater than or equal to the current number of slots, the method creates additional parking slots to match the desired number. Each new parking slot is created with a default size of 0, but you can adjust the size as needed.

If the number_of_slots is less than the current number of slots, the method destroys the excess parking slots to match the desired number. The destroy_all method is used with a limit to specify the number of slots to be destroyed.

Model: Entrance
# app/models/entrance.rb
class Entrance < ApplicationRecord
    belongs_to :parking_space
    has_many :parking_slots, dependent: :nullify
end
The Entrance model represents an entrance to a parking space. It belongs to a ParkingSpace and has many associated ParkingSlots. The dependent: :nullify option ensures that if an entrance is deleted, the associated parking slots will have their entrance_id set to NULL instead of being deleted.

Model: ParkingSlot
# app/models/parking_slot.rb
class ParkingSlot < ApplicationRecord
    belongs_to :parking_space
    belongs_to :entrance, optional: true
    belongs_to :vehicle, optional: true
    attribute :size, :integer, default: 0
    enum size: { small: 0, medium: 1, large: 2 }
  
    # Update scope to consider both no vehicle and specific size
    scope :available, ->(size) { where(vehicle_id: nil, size: sizes[size]) }
end
The ParkingSlot model represents a parking slot within a parking space. It belongs to a ParkingSpace, an Entrance, and a Vehicle. The optional: true option is used for the entrance and vehicle associations to allow them to be NULL if not present.

The model also defines an attribute size with a default value of 0. It uses the enum feature to define the possible sizes of a parking slot as small, medium, and large.

The model includes a scope named available that takes a size parameter. This scope is used to retrieve available parking slots based on the specified size. It filters the parking slots by checking if the vehicle_id is NULL and the size matches the specified size.

Model: Vehicle
# app/models/vehicle.rb
```
class Vehicle < ApplicationRecord
    enum size: { small: 0, medium: 1, large: 2 }
    belongs_to :parking_slot, optional: true
    attr_accessor :slot_id
end
The Vehicle model represents a vehicle that can be parked in a parking slot. It uses the enum feature to define the possible sizes of a vehicle as small, medium, and large.

The model belongs to a ParkingSlot and has an optional association with it. The attr_accessor is used to define a virtual attribute slot_id, which can be used to assign a parking slot to a vehicle.

Controller: ParkingController
# app/controllers/parking_controller.rb
class ParkingController < ApplicationController
    SLOT_SIZES = { SP: :small, MP: :medium, LP: :large }.freeze
    BASE_RATE = 40
    EXCEEDING_RATES = { SP: 20, MP: 60, LP: 100 }.freeze
    FULL_DAY_FEE = 5000

    def assign_slot(vehicle)
        available_slots = vehicle.available_parking_slots
        vehicle_size = vehicle.size.to_sym
    
        available_slots.each do |slot|
          slot_size = slot.size.to_sym
          return slot if can_park_in_slot?(vehicle_size, slot_size)
        end
    
        nil
      end
    
    def can_park_in_slot?(vehicle_size, slot_size)
        case vehicle_size
            when :small
                return true if %i[SP MP LP].include?(slot_size)
            when :medium
                return true if %i[MP LP].include?(slot_size)
            when :large
                return true if slot_size == :LP
        end
        false
    end

    def calculate_fee(parked_time, slot_size)
        base_fee = BASE_RATE * [parked_time, 3].min
        exceeding_hours = [parked_time - 3, 0].max
        exceeding_fee = exceeding_hours * EXCEEDING_RATES[slot_size]
    
        total_fee = base_fee + exceeding_fee
        total_fee += FULL_DAY_FEE * (parked_time / 24).floor if parked_time >= 24
    
        total_fee.ceil
    end
  
  
    def park(vehicle)
        slot = assign_slot(vehicle)
        if slot.present?
            vehicle.update(parking_slot: slot)
            { confirmation: 'Successfully parked', slot: slot.id }
        else
            { error: 'No available slots' }
        end
      end
  
      def unpark(vehicle)
        if vehicle.parking_slot.present?
            parked_time = calculate_parked_time(vehicle)
            if parked_time.nil?
                { error: 'Error calculating parked time' }
            else
                slot_size = SLOT_SIZES[vehicle.parking_slot.size.to_sym]
                fee = calculate_fee(parked_time, slot_size)
                vehicle.update(parking_slot: nil)
                { confirmation: 'Successfully exited', fee: fee }
            end
        else
            { error: 'Vehicle is not parked' }
        end
      end
  
    private
  
    def calculate_parked_time(vehicle)
        parked_time = (Time.now - vehicle.parking_slot.created_at) / 3600 rescue nil # Convert to hours
    end
end
```
The ParkingController handles the parking and unparking of vehicles. It includes several methods to perform these actions.

The assign_slot method takes a vehicle as a parameter and assigns an available parking slot to it. It retrieves the available parking slots for the vehicle using the available_parking_slots method. It then iterates over the available slots and checks if the vehicle can be parked in each slot by calling the can_park_in_slot? method. If a suitable slot is found, it is returned. Otherwise, nil is returned.

The can_park_in_slot? method takes the vehicle_size and slot_size as parameters and checks if the vehicle can be parked in the slot based on their sizes. It uses a case statement to handle different vehicle sizes and slot sizes. If the vehicle size matches the slot size, it returns true. Otherwise, it returns false.

The calculate_fee method takes the parked_time and slot_size as parameters and calculates the parking fee based on the parked time and slot size. It calculates the base fee by multiplying the BASE_RATE with the minimum of the parked time and 3. It then calculates the exceeding hours by subtracting 3 from the parked time and taking the maximum of the result and 0. The exceeding fee is calculated by multiplying the exceeding hours with the rate specified in the EXCEEDING_RATES hash for the given slot size.

The total fee is calculated by adding the base fee and the exceeding fee. If the parked time is greater than or equal to 24 hours, an additional fee is added for each full day of parking. The total fee is then rounded up to the nearest integer.

The park method takes a vehicle as a parameter and parks the vehicle in an available slot. It calls the assign_slot method to get an available slot for the vehicle. If a slot is found, the vehicle's parking_slot association is updated, and a confirmation message with the slot ID is returned. If no slot is available, an error message is returned.

The unpark method takes a vehicle as a parameter and unparks the vehicle from its parking slot. It calculates the parked time using the calculate_parked_time method. If the parked time is successfully calculated, the slot size is determined based on the vehicle's parking slot size using the SLOT_SIZES hash. The parking fee is calculated using the calculate_fee method. The vehicle's parking_slot association is then updated to nil, and a confirmation message with the fee is returned. If the parked time cannot be calculated, an error message is returned. If the vehicle is not parked, an error message is returned.

Controller: EntrancesController
# app/controllers/entrances_controller.rb
```
class EntrancesController < ApplicationController
    def new
        @entrance = Entrance.new
        @parking_spaces = ParkingSpace.all # Fetch all parking spaces for selection
    end
  
    def create
        @entrance = Entrance.new(entrance_params)
        if @entrance.save
            redirect_to @entrance.parking_space, notice: 'Entrance added successfully'
        else
            @parking_spaces = ParkingSpace.all
            flash.now[:alert] = 'Failed to create entrance. Please check the provided information.'
            render :new
        end
    end
  
    private
  
    def entrance_params
        params.require(:entrance).permit(:name, :location, :parking_space_id)
    end
end
```
The EntrancesController handles the creation of entrances for parking spaces. It includes two methods: new and create.

The new method initializes a new Entrance object and fetches all parking spaces from the database to populate a selection dropdown in the view.

The create method creates a new Entrance object with the permitted parameters from the form. If the entrance is successfully saved, the user is redirected to the associated parking space's show page with a success notice. If the entrance fails to save, the user is rendered the new view again with an error message.

Controller: ParkingSpacesController
# app/controllers/parking_spaces_controller.rb
```
class ParkingSpacesController < ApplicationController
    def index
        @parking_spaces = ParkingSpace.includes(parking_slots: :vehicle).all
    end
      
    def edit_slots
        @parking_space = ParkingSpace.find(params[:id])
    end
  
    def update_slots
        @parking_space = ParkingSpace.find(params[:id])
        if @parking_space.update(parking_space_params)
          @parking_space.adjust_slots(params[:parking_space][:number_of_slots].to_i)
          redirect_to @parking_space, notice: 'Parking slots updated successfully'
        else
          render :edit_slots
        end
      end
  
    def park_manually(vehicle, size, slot_id)
        slot = ParkingSlot.find(slot_id)
        vehicle.size = size
    
        if slot.present? && vehicle.valid? && vehicle.save
          if slot.update(vehicle_id: vehicle.id, size: vehicle.size)
            { confirmation: 'Successfully parked manually', slot: slot.id }
          else
            { error: 'Failed to manually park the vehicle' }
          end
        else
          { error: 'Failed to manually park the vehicle: Invalid size or attributes' }
        end
      end
  
    def unpark_and_view_fee
        vehicle = Vehicle.find_by(id: params[:vehicle_id])
        if vehicle.present? && vehicle.parking_slot.present?
            parked_time = calculate_parked_time(vehicle)
            if parked_time.nil?
                return { error: 'Error calculating parked time' }
            else
                fee = calculate_fee(vehicle, parked_time)
                return { vehicle: vehicle, fee: fee }
            end
        else
            return { error: 'Vehicle is not parked' }
        end
    end
    
    def delete_vehicle
        vehicle = Vehicle.find_by(id: params[:vehicle_id])
        if vehicle.present? && vehicle.parking_slot.present?
            if vehicle.update(parking_slot: nil)
                return { confirmation: 'Successfully exited' }
            else
                return { error: 'Failed to exit the vehicle' }
            end
        else
            return { error: 'Vehicle is not parked' }
        end
    end

    def new_manual_parking
        @slot_id = params[:id]
        @vehicle = Vehicle.new
    end
  
    def create_manual_parking
        puts "Received parameters: #{params.inspect}"
        vehicle = Vehicle.new(vehicle_params)
        slot_id = params.dig(:vehicle, :slot_id)
        vehicle.parking_slot_id = slot_id
    
        if vehicle.save
            result = park_manually(vehicle, vehicle.size, slot_id)
            if result[:confirmation]
                redirect_to root_path, notice: result[:confirmation]
            else
                flash.now[:alert] = result[:error]
                render :new_manual_parking
            end
        else
            flash.now[:alert] = 'Failed to create a vehicle'
            render :new_manual_parking
        end
    end

    private
  
    def vehicle_params
        params.require(:vehicle).permit(:size, :slot_id)
    end
    
    def calculate_parked_time(vehicle)
        parked_time = (Time.now - vehicle.parking_slot.created_at) / 3600 rescue nil # Convert to hours
    end

    def calculate_fee(vehicle, parked_time)
        base_rate = 40
        exceeding_rates = { small: 20, medium: 60, large: 100 }
        size_sym = vehicle&.size&.to_sym || :small
        exceeding_hours = [parked_time - 3, 0].max
        base_fee = base_rate * [parked_time, 3].min
        exceeding_fee = exceeding_hours * exceeding_rates[size_sym]
        total_fee = base_fee + exceeding_fee
      
        if parked_time >= 24
            total_fee = ((parked_time / 24).floor * 5000) + (total_fee % 5000)
        end
      
        total_fee.ceil
    end

    def parking_space_params
        params.require(:parking_space).permit(:number_of_slots)
    end
end
```
The ParkingSpacesController handles the management of parking spaces. It includes several methods to perform various actions related to parking spaces.

The index method retrieves all parking spaces from the database, including their associated parking slots and vehicles. This is done using the includes method to eager load the associations and avoid N+1 queries.

The edit_slots method finds the parking space with the specified ID and assigns it to the @parking_space instance variable. This is used to display the edit slots form in the view.

The update_slots method finds the parking space with the specified ID and updates its attributes based on the permitted parameters from the form. It then calls the adjust_slots method to adjust the number of parking slots based on the provided number_of_slots parameter. If the parking space is successfully updated and the slots are adjusted, the user is redirected to the parking space's show page with a success notice. If there are any errors, the user is rendered the edit_slots view again.

The park_manually method takes a vehicle, size, and slot_id as parameters and manually parks the vehicle in the specified slot. It first finds the parking slot with the specified ID. It then assigns the vehicle's size based on the provided size parameter. If the slot is present, the vehicle is valid, and the vehicle is successfully saved, the parking slot is updated with the vehicle's ID and size. A confirmation message with the slot ID is returned if the parking is successful. Otherwise, an error message is returned.

The unpark_and_view_fee method retrieves the vehicle with the specified ID and checks if it is present and parked. If the vehicle is parked, the parked time is calculated using the calculate_parked_time method. If the parked time is successfully calculated, the parking fee is calculated using the calculate_fee method. The vehicle and fee are returned as a result. If there are any errors, an error message is returned.

The delete_vehicle method retrieves the vehicle with the specified ID and checks if it is present and parked. If the vehicle is parked, its parking slot is updated to nil, indicating that it has exited the parking space. A confirmation message is returned if the vehicle is successfully exited. Otherwise, an error message is returned.

The new_manual_parking method assigns the specified slot ID and initializes a new Vehicle object. This is used to display the manual parking form in the view.

The create_manual_parking method creates a new Vehicle object with the permitted parameters from the form. It assigns the parking slot ID based on the provided slot_id parameter. If the vehicle is successfully saved, the park_manually method is called to manually park the vehicle in the specified slot. If the parking is successful, the user is redirected to the root path with a success notice. If there are any errors, the user is rendered the new_manual_parking view again with an error message.

Views: index.html.erb
# app/views/parking_spaces/index.html.erb
```
<h1>Parking Information</h1>
<table border="1">
    <thead>
        <tr>
            <th>Parking Space ID</th>
            <th>Parking Slot ID</th>
            <th>Vehicle ID</th>
            <th>Vehicle Size</th>
            <th>Action</th>
        </tr>
    </thead>
    <tbody>
        <% @parking_spaces.each do |space| %>
          <% space.parking_slots.each do |slot| %>
            <tr>
              <td><%= space.id %></td>
              <td><%= slot.id %></td>
              <td><%= slot.vehicle_id %></td>
              <td><%= slot.vehicle&.size %></td>
              <td>
                <% if slot.vehicle.present? %>
                  <% parked_time = calculate_parked_time(vehicle) %>
                  <% if parked_time.present? %>
                    <% fee = calculate_fee(slot.vehicle, parked_time) %>
                    <%= fee %>
                    <%= link_to 'Pay & Unpark', delete_vehicle_path(vehicle_id: slot.vehicle.id), method: :get, data: { confirm: 'Are you sure?' } %>
                  <% else %>
                    <%= link_to 'Park Manually', new_manual_parking_path(slot.id) %>
                  <% end %>
                <% end %>
              </td>
            </tr>
          <% end %>
        <% end %>
    </tbody>
</table>
```
The index.html.erb view displays the parking information for all parking spaces. It includes a table with columns for the parking space ID, parking slot ID, vehicle ID, vehicle size, and an action column.

The view iterates over each parking space and its associated parking slots using nested loops. For each parking slot, it displays the parking space ID, parking slot ID, vehicle ID, and vehicle size in separate table cells. It also includes an action column that provides options to pay and unpark the vehicle or manually park a vehicle if it is not already parked.

The view uses the calculate_parked_time and calculate_fee methods to calculate the parked time and fee for each parking slot's vehicle. It conditionally displays the fee and provides links to pay and unpark the vehicle or park manually based on the availability of the vehicle and the calculated parked time.

Views: edit_slots.html.erb
# app/views/parking_spaces/edit_slots.html.erb
```
<h1>Edit Slots for Parking Space ID: <%= @parking_space.id %></h1>
<%= form_with(model: @parking_space, url: update_slots_parking_space_path(@parking_space), method: :patch) do |form| %>
    <%= form.label :number_of_slots %>
    <%= form.number_field :number_of_slots %>
    <%= form.submit 'Update Slots' %>
<% end %>
```
The edit_slots.html.erb view displays a form to edit the number of slots for a parking space. It includes a heading with the parking space ID and a form to update the number of slots.

The view uses the form_with helper to create a form for the @parking_space object. The form is submitted to the update_slots action of the ParkingSpacesController using the update_slots_parking_space_path route helper.

The form includes a label and number field for the number_of_slots attribute of the parking space. It also includes a submit button to update the slots.

Views: new.html.erb
# app/views/entrances/new.html.erb
```
<h1>Manual Vehicle Parking</h1>
<%= form_with(model: @vehicle, url: create_manual_parking_path, method: :post) do |form| %>
    <%= form.hidden_field :slot_id, value: @slot_id %>
    <%= form.label :size %>
    <%= form.select :size, Vehicle.sizes.keys %>
    <%= form.submit 'Park Vehicle' %>
<% end %>
```
The new.html.erb view displays a form for manual vehicle parking. It includes a heading and a form to select the vehicle size and park the vehicle manually.

The view uses the form_with helper to create a form for the @vehicle object. The form is submitted to the create_manual_parking action of the ParkingSpacesController using the create_manual_parking_path route helper.

The form includes a hidden field for the slot_id parameter, which is set to the @slot_id value. This allows the slot ID to be passed to the controller when submitting the form.

The form includes a label and select field for the size attribute of the vehicle. The select field options are generated based on the possible sizes defined in the Vehicle model.

The form also includes a submit button to park the vehicle manually.

Routes: routes.rb
Rails.application.routes.draw do
```
  root 'parking_spaces#index'

  get '/unpark_and_view_fee', to: 'parking#unpark_and_view_fee', as: :unpark_and_view_fee
  get '/delete_vehicle', to: 'parking#delete_vehicle', as: :delete_vehicle

  get 'manual_parking/:id', to: 'parking_spaces#new_manual_parking', as: 'new_manual_parking'
  post 'manual_parking', to: 'parking_spaces#create_manual_parking', as: 'create_manual_parking'

  resources :entrances, only: [:new, :create]
end
```
The routes.rb file defines the routes for the application. It includes routes for the root path, unparking and viewing the fee, deleting a vehicle, and manual parking.

The root path is set to the index action of the ParkingSpacesController, which displays the parking information for all parking spaces.

The unpark_and_view_fee route is defined as a GET request to the unpark_and_view_fee action of the ParkingController. It is named unpark_and_view_fee using the as option.

The delete_vehicle route is defined as a GET request to the delete_vehicle action of the ParkingController. It is named delete_vehicle using the as option.

The new_manual_parking route is defined as a GET request to the new_manual_parking action of the ParkingSpacesController. It includes a parameter :id to specify the slot ID. It is named new_manual_parking using the as option.

The create_manual_parking route is defined as a POST request to the create_manual_parking action of the ParkingSpacesController. It is named create_manual_parking using the as option.

The entrances resource is defined with only the new and create actions. These routes are used for creating entrances for parking spaces.

seeds.rb
# seeds.rb
```
# Clear existing data
Vehicle.delete_all
ParkingSlot.delete_all
Entrance.delete_all
ParkingSpace.delete_all

# Create parking spaces and adjust slots
3.times do
  space = ParkingSpace.create
  space.adjust_slots(number_of_slots: 5) # Adjust the number of slots for each space
end

# Create sample entrances for each parking space
parking_spaces = ParkingSpace.all
parking_spaces.each_with_index do |space, index|
  Entrance.create(name: "Entrance #{('A'.ord + index).chr}", location: "Location #{('A'.ord + index).chr}", parking_space_id: space.id)
end

# Create vehicles and associate them with parking slots (example)
parking_slots = ParkingSlot.limit(10) # Retrieve some slots to assign vehicles
parking_slots.each_with_index do |slot, index|
  size = index % 3 # Just an example, modify based on your needs
  vehicle = Vehicle.create(size: size, parking_slot_id: slot.id)
  slot.update(vehicle_id: vehicle.id) # Associate vehicle with parking slot
end
```
The seeds.rb file is used to populate the database with sample data. It clears any existing data for the Vehicle, ParkingSlot, Entrance, and ParkingSpace models.

It then creates three parking spaces and adjusts the number of slots for each space to 5 using the adjust_slots method.

Next, it creates sample entrances for each parking space. The name and location of each entrance are generated based on the index of the parking space.

Finally, it creates vehicles and associates them with parking slots. It retrieves some parking slots using the limit method and assigns a size to each vehicle based on the index. The parking slot is then updated with the vehicle's ID to associate them.
